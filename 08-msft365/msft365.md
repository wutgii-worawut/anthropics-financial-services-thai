---
title: Microsoft 365 Add-in
parent: 08 Microsoft 365 Add-in
nav_order: 1
---

# Microsoft 365 Add-in สำหรับ Claude

## ทำอะไร

Plugin `claude-for-msft-365-install` เป็นเครื่องมือสำหรับ IT administrator ของบริษัท — ใช้ provision Claude ให้ทำงานภายใน Excel, PowerPoint, Word, และ Outlook ที่ tenant ระดับ organization

แทนที่จะให้ Claude ของบริษัทพูดคุยกับ Anthropic API โดยตรง plugin นี้สามารถ redirect ไปใช้ **cloud backend ของคุณเอง** — Google Vertex AI, AWS Bedrock, หรือ internal LLM gateway ที่บริษัทสร้างขึ้น

## ผู้ใช้งาน

**Microsoft 365 admin** — คนที่มี access ดำเนิน Azure admin console และสามารถยิง PowerShell scripts ได้

หลังจาก admin setup เสร็จ end-user ก็ใช้ Claude จากใน Office app โดยไม่ต้องรู้ backend ทำงานยังไง

## โครงสร้าง

Plugin มี 4 ส่วนหลัก:

```
claude-for-msft-365-install/
├── commands/                    # slash commands สำหรับ CLI (Claude Code)
│   ├── setup.md                 # interactive wizard สำหรับ provisioning
│   ├── manifest.md              # generate add-in XML manifest
│   ├── consent.md               # produce Azure admin consent URL
│   ├── bootstrap.md             # build per-user config endpoint
│   └── update-user-attrs.md     # write config ไปยัง Microsoft Graph
├── scripts/
│   └── build-manifest.mjs       # Node.js helper สำหรับ manifest generation
└── examples/
    └── python-bootstrap/        # reference Python app สำหรับ bootstrap endpoint
        ├── app.py               # Flask server ที่ emit per-user MCP + skills
        ├── config.py            # environment setup
        └── requirements.txt
```

## ขั้นตอน Install

Plugin นี้ไม่ได้ใช้ Cowork — ใช้ Claude Code CLI แทน:

```bash
claude plugin install claude-for-msft-365-install@financial-services-plugins
```

จากนั้นใช้ slash command:

```bash
/claude-for-msft-365-install:setup
```

### มีอะไรเกิดขึ้นภายใน

1. **`/setup`** — interactive wizard ถาม:
   - Cloud backend type (Vertex AI / Bedrock / custom gateway)
   - Resource identifiers (project ID, region, endpoint URL)
   - Azure tenant ID
   - Microsoft 365 app registration details

2. **`/manifest`** — generate customized XML manifest ที่ Office recognizes ที่บ้าน:
   - embed cloud endpoint URL
   - configure OAuth scopes
   - set security properties

3. **`/consent`** — ให้ link ไป Azure consent screen
   - Microsoft 365 admin ปล่อย "grant consent" สำหรับ app registration
   - เป็นการให้สิทธิ์ plugin ที่จะ read user graph profile (email, name, ...)

4. **`/bootstrap`** — สร้าง endpoint ที่ Office add-in เรียก ทุกครั้งที่ user เปิด Excel/Word
   - endpoint return per-user:
     - MCP server URLs (connect ไปที่ firm's data)
     - list of enabled skills
     - dynamic Claude instructions (เช่น "use company templates")
   - ดูตัวอย่าง Python app ใน `examples/python-bootstrap/`

5. **`/update-user-attrs`** — write per-user config via Microsoft Graph extension attributes
   - เช่น "alice@firm.com ได้ access ไปยัง Investment Banking skills เท่านั้น"

## ตัวอย่าง: Excel + DCF Model

User เปิด Excel ที่มี blank DCF template — Claude sidebar ปรากฏ

```
Claude for Excel

User: "Fill in the WACC calculation for a 10-year hospital operator"

Claude: 
[ดึง underlying cost-of-capital data จาก MCP connector]
[fill cells B2:B10 ด้วย assumptions]
[refresh formula tree]

Done. WACC = 8.2% (see sheet WACC_calc)
```

ทำได้เพราะ:
- Office add-in ทำให้ Claude read/write Excel cells
- `skills` ที่ installed (จาก bootstrap) มี `dcf-model` ← ขั้นตอน WACC calculation
- MCP connector ดึง Treasury yield curve จาก firm's data provider

## ความปลอดภัย

### Admin Consent

- Microsoft 365 admin ต้อง explicitly approve app ใน Azure
- "Consent" = "I give this app permission to call Microsoft Graph on behalf of users"
- ไม่ delegate capability — แค่ authenticate

### OAuth Scopes

- Manifest list ทุก scope ที่ add-in ขอ
- เทพฟ์: `.default` (read profile), Office document scope (read/write cell/slide)
- ไม่ include mail, calendar, Teams — administrator ควรตรวจสอบ

### Data Flow

```
User opens Excel
↓
Office calls bootstrap endpoint (with user's Azure token)
↓
Bootstrap returns: {
  mcp_servers: [ {url: "...", credentials: {...}} ],
  skills: ["dcf-model", "comps-analysis"],
  system_instruction: "You are a financial analyst..."
}
↓
Claude calls /v1/messages with skills + MCP servers loaded
↓
User query → Claude calls skill → MCP reads firm data
↓
Result written back to Excel cells (via Office API)
```

### Secret Management

- Cloud backend credentials (Vertex AI key, Bedrock role, gateway token) ไม่ embed ใน manifest
- Bootstrap endpoint เก็บมันใน env var หรือ vault
- ทุกครั้ง user request → bootstrap ตรวจสอบ permission ก่อน emit MCP server URL
- "alice ได้เข้า Bloomberg data?" — เช็คใน directory, ไม่ emit Bloomberg MCP ถ้าเธอไม่มี license

## Command Reference

| Command | What it does |
|---|---|
| `/claude-for-msft-365-install:setup` | Step-by-step wizard — cloud resources, admin consent, manifest + bootstrap config |
| `/claude-for-msft-365-install:manifest` | Generate the customized Office add-in XML — copy paste ไปใน Azure app registration |
| `/claude-for-msft-365-install:consent` | Azure admin consent URL — ให้ Microsoft 365 admin click และ approve |
| `/claude-for-msft-365-install:bootstrap` | Show the bootstrap endpoint code — deploy ไปยัง your server |
| `/claude-for-msft-365-install:update-user-attrs` | Write per-user skill + MCP allowlist via Microsoft Graph |

---

**Next:** [Deployment guide (Section 09)](../09-deployment) — วิธี install plugins ใน Cowork, Claude Code, และ Managed Agents API
