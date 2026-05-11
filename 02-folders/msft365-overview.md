---
title: Claude for Microsoft 365 — Installation Tooling
parent: 02 โครงสร้างโฟลเดอร์
nav_order: 7
---

# Claude for Microsoft 365 Installation

ที่อยู่: `claude-for-msft-365-install/`

นี่คือ **admin tooling plugin** — **แยกต่างหาก** จากตัวแทน (agents) และ vertical plugins — สำหรับองค์กรที่ต้องการ deploy Claude บน Excel, PowerPoint, Word, Outlook ผ่าน **cloud ของตัวเอง** (Vertex AI, Bedrock, หรือ gateway)

## ทำไมแยกจากส่วนอื่น

### 1. ติดตั้งเป้าหมาย (Install Target) ต่างกัน

| Plugins | Install Target | Usage |
|---|---|---|
| agents, verticals, partner-built | Claude.com Cowork หรือ `/v1/agents` API | Analysts use agents directly |
| **claude-for-msft-365-install** | **Microsoft 365 add-in manifest** | IT admins configure tenant |

### 2. Audience ต่างกัน

- Agents/verticals → **Financial analysts, traders, advisors**
- claude-for-msft-365-install → **IT administrators, security, cloud engineers**

### 3. Deployment Model ต่างกัน

**Agents/verticals**: User installs → works immediately
**MSFT 365 add-in**: Admin provisions → enables all users in org

## ลักษณะการทำงาน

### ปัญหา

องค์กรที่ใช้ Microsoft 365 แต่:
- ไม่สามารถใช้ Anthropic API โดยตรง (data residency, compliance)
- ต้องส่ง inference ไป **internal LLM gateway** หรือ **Vertex AI / Bedrock** ใน organization's own cloud

### วิธีแก้

`claude-for-msft-365-install` = **interactive wizard** ที่:
1. กำหนด cloud destination (Vertex AI, Bedrock, custom gateway)
2. สร้าง Azure app registration + สิทธิ์
3. Generate customized **add-in manifest XML**
4. Write per-user routing config ผ่าน **Microsoft Graph extension attributes**

## Commands

### `/claude-for-msft-365-install:setup`

**Interactive wizard** — ทำ end-to-end provisioning:

```
Step 1: What's your cloud provider?
  → Vertex AI (Google Cloud)
  → Bedrock (AWS)
  → Custom LLM gateway (internal)

Step 2: Cloud credentials
  → [Enter API key / service account / OAuth token]

Step 3: Organization info
  → Tenant ID: <Azure tenant>
  → User domain: @example.com

Step 4: Review & deploy
  → Creates app registration
  → Generates manifest
  → Writes Graph config
  → Done!
```

### `/claude-for-msft-365-install:manifest`

**Generate manifest XML** (rerun หากต้องปรับแต่ง):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<OfficeApp xmlns="http://schemas.microsoft.com/office/appforoffice/1.1">
  <Id>12345678-1234-1234-1234-123456789012</Id>
  <Version>1.0.0.0</Version>
  <ProviderName>Claude for Financial Services</ProviderName>
  <DefaultLocale>en-US</DefaultLocale>
  <DisplayName DefaultValue="Claude Financial Services"/>
  <Description DefaultValue="Investment banking, research, and ops agents"/>
  <Hosts>
    <Host Name="Document"/>
    <Host Name="Workbook"/>
    <Host Name="Presentation"/>
    <Host Name="Mailbox"/>
  </Hosts>
  <DefaultRibbon>
    <Ribbon Id="TabHome">
      <Group Id="ClaudeGroup">
        <Label resid="groupLabel"/>
        <Icon>
          <Image size="16" src="..."/>
        </Icon>
        <Control xsi:type="Button" id="invokeClaudeButton">
          <Label resid="buttonLabel"/>
          <Supertip>
            <Title resid="buttonTitle"/>
            <Description resid="buttonDescription"/>
          </Supertip>
          <Action xsi:type="ExecuteFunction">
            <FunctionName>handleClaudeInvoke</FunctionName>
          </Action>
        </Control>
      </Group>
    </Ribbon>
  </DefaultRibbon>
  <Runtime resid="Taskpane.Url"/>
  <Permissions>ReadWriteDocument</Permissions>
</OfficeApp>
```

### `/claude-for-msft-365-install:consent`

**Azure admin consent flow URL**:

```
https://login.microsoftonline.com/.../oauth2/v2.0/authorize
  ?client_id=...
  &scope=Directory.ReadWrite
  &...
```

Admin เปิด URL → ให้ consent → add-in registered in tenant

### `/claude-for-msft-365-install:update-user-attrs`

**Write per-user routing** via Microsoft Graph extension attributes:

```bash
# For each user:
PATCH /users/alice@example.com/extensions/claude-config
{
  "cloud_provider": "vertex-ai",
  "cloud_project": "my-org-project",
  "cloud_region": "us-central1",
  "gateway_url": "https://gateway.internal.example.com/claude",
  "allowed_agents": ["pitch-agent", "model-builder"],
  "data_residency": "US"
}
```

ผลลัพธ์: เมื่อ Alice เปิด Excel + ใช้ Claude → ส่ง ไป Vertex AI project ของ org แล้ว

### `/claude-for-msft-365-install:bootstrap`

**Build bootstrap endpoint** — per-user MCP servers, skills, dynamic config:

```
bootstrap.example.com/users/alice@example.com
  → Returns: {
    cloud_config: {...},
    available_agents: [pitch-agent, gl-reconciler],
    mcp_servers: [daloopa, morningstar, ...],
    skills: [...],
    system_context: "..."
  }
```

Add-in ใน Excel fetches bootstrap ก่อนเป็นครั้งแรก → loads config dynamically

## Architecture: Add-in vs Agents

```
User opens Excel
  ↓
Claude for Microsoft 365 add-in loads
  ↓
Calls /claude-for-msft-365-install:bootstrap endpoint
  → Fetches cloud config + agents ที่ user มี permission
  ↓
User clicks "Analyze earnings..." in sidebar
  ↓
Add-in dispatches ไป cloud LLM (Vertex AI / Bedrock / gateway)
  → NOT ไป Anthropic API
  ↓
Behind cloud LLM: agents + skills ทำงาน
  (same earnings-reviewer, market-researcher, model-builder agents from this repo)
  ↓
Results returned → displayed in Excel sidebar
```

## การเชื่อมต่อกับ Agents + Verticals

```
Agents repo:
├── plugins/agent-plugins/pitch-agent/
│   ├── agents/pitch-agent.md       ← system prompt
│   └── skills/...                  ← bundled skills
├── plugins/vertical-plugins/financial-analysis/
│   └── .mcp.json                   ← connectors (Daloopa, Morningstar, ...)
└── managed-agent-cookbooks/        ← for API deployment

                       ↑ shared
                       │
Claude for MSFT 365:
├── commands/
├── .mcp.json                       ← can add more connectors (SAP, Oracle, ...)
└── bootstrap logic                 ← serves agents + config to add-in
```

ผลลัพธ์: **agents + skills เหมือนกัน** — Cowork + API + Office ใช้ core logic เดียวกัน

## Configuration Files

ไม่มี source code — pure configuration:

```
claude-for-msft-365-install/
├── .claude-plugin/
│   └── plugin.json         # Claude Code plugin metadata
├── commands/
│   ├── setup.md            # /setup wizard
│   ├── manifest.md         # /manifest generator
│   ├── consent.md          # /consent URL builder
│   ├── update-user-attrs.md # /update-user-attrs bulk tool
│   └── bootstrap.md        # /bootstrap endpoint setup
└── README.md               # Admin guide
```

## Installation

```bash
# In Claude Code session:
claude plugin marketplace add anthropics/claude-for-financial-services
claude plugin install claude-for-msft-365-install@claude-for-financial-services

# Then in session:
/claude-for-msft-365-install:setup
```

## Security Considerations

### Organization Control

- Data stays in **organization's cloud** (Vertex AI, Bedrock, gateway)
- Anthropic **never sees** financial data flowing through Excel
- Admin controls **who can use which agents** (extension attributes)

### Tenant Isolation

- App registration ต่อ tenant หนึ่ง
- Cannot be sideloaded into another org without new consent

### Token & Credential Management

- Cloud credentials stored ใน Azure Key Vault (or org's secret manager)
- add-in gets temporary tokens from bootstrap endpoint
- Tokens rotated per request

## Comparison Table

| Aspect | Cowork | Managed Agents API | MSFT 365 |
|---|---|---|---|
| **Entry point** | claude.com/product/cowork | `/v1/agents` → orchestrator | Excel/PowerPoint/Word/Outlook |
| **Installation** | User clicks add | Engineer deploys + orchestrates | Admin provisions |
| **Data location** | Anthropic cloud | Customer's event bus | Org's cloud (Vertex/Bedrock/gateway) |
| **Agents/skills** | Same plugins | Same cookbooks | Same agents + skills |
| **Auth** | User account | API key | OAuth to org cloud |
| **Best for** | Individual analysts | Automation flows | Enterprise Office users |

---

**สรุป**: 
- `claude-for-msft-365-install` คือ **admin tooling** — **ไม่ใช่ agent**
- ให้ organizations deploy Claude agents ใน Office apps โดยไม่ผ่าน Anthropic cloud
- ใช้ **agents + skills เดียวกัน** กับ Cowork + API
- Wizard + bootstrap endpoints ทำให้ provisioning ใน Azure + Microsoft 365 อัตโนมัติ
