---
title: Quick Start
parent: 01 ภาพรวม
nav_order: 4
---

# Quick Start

Claude for Financial Services พร้อมเลือกให้ 3 วิธี: web UI (Cowork), CLI (Claude Code), หรือ headless (Managed Agents API)

## Prerequisites

ทั้ง 3 วิธี ต้องมี:
- **Claude API key** — `sk-ant-...` (ได้จาก [console.anthropic.com](https://console.anthropic.com))
- **Anthropic account** — เข้าถึง API documentation

ตัวเลือก:
- **MCP provider credentials** — Daloopa, CapIQ, FactSet, Moody's, ฯลฯ (ใช้เต็มความสามารถ)
- **Internal MCP server** — สำหรับ Managed Agents (firm-specific GL, subledger, CRM data)

---

## วิธี 1: Cowork Plugin (Web UI) — ง่ายที่สุด

ไม่ต้องติดตั้งอะไร — login เข้า [claude.com](https://claude.com) → Settings → Plugins

### ขั้นตอน 1: Add Repository

```
Settings → Plugins → Add plugin
↓
Paste URL: https://github.com/anthropics/claude-for-financial-services
↓
Browse marketplace for agents and verticals
```

(README.md line 51–55)

### ขั้นตอน 2: Select Agents/Verticals

Marketplace แสดง:
- **10 Named Agents** — Pitch Agent, Market Researcher, GL Reconciler, KYC Screener, ฯลฯ
- **7 Verticals** — financial-analysis (core), investment-banking, equity-research, private-equity, wealth-management, fund-admin, operations

เลือกตามสิ่งที่ต้อง:
- **IB bank**: Pitch Agent + investment-banking vertical
- **Equity research**: Market Researcher + equity-research vertical
- **PE firm**: KYC Screener + private-equity vertical
- **Fund admin**: GL Reconciler + fund-admin vertical
- **Wealth manager**: wealth-management vertical

### ขั้นตอน 3: ลองใช้

หลังติดตั้ง slash commands พร้อมใช้:

```
/comps —— Comparable company analysis
/dcf —— DCF valuation with WACC
/lbo —— Leveraged buyout model
/cim —— Confidential Information Memorandum
/earnings —— Post-earnings update
/ic-memo —— Investment committee memo
/portfolio —— Portfolio monitoring
/financial-plan —— Financial planning
... ดูเพิ่มเติมใน /commands
```

(README.md line 160–213)

**ตัวอย่าง: ลองครั้งแรก**

```
สวัสดี Claude

ฉันต้องการวิเคราะห์ comps สำหรับ Acme Corporation (ACME).
ปัจจุบัน trading at $150/share, market cap $3B.

ช่วยฉันได้ไหม?
```

Claude จะเลือก `/comps` command โดยอัตโนมัติ และ:
1. Query MCP servers (Daloopa, CapIQ, FactSet) สำหรับ peer set
2. ดึง trading multiples (EV/EBITDA, P/E, EV/Sales, ฯลฯ)
3. Draft summary table + outlier flags
4. Suggest valuation range (low/median/high)

---

## วิธี 2: Claude Code CLI — สำหรับ Developers

ต้องมี Claude Code CLI installed ([install.claudecode.com](https://install.claudecode.com))

### ขั้นตอน 1: Add Marketplace

```bash
claude plugin marketplace add anthropics/claude-for-financial-services
```

### ขั้นตอน 2: Install Core Plugin

```bash
# ต้องติดตั้งก่อนเสมอ — มี shared skills + MCP configs
claude plugin install financial-analysis@claude-for-financial-services
```

### ขั้นตอน 3: Install Agents & Verticals

ติดตั้งเฉพาะที่ต้อง:

```bash
# Named agents
claude plugin install pitch-agent@claude-for-financial-services
claude plugin install market-researcher@claude-for-financial-services
claude plugin install gl-reconciler@claude-for-financial-services
claude plugin install earnings-reviewer@claude-for-financial-services
claude plugin install kyc-screener@claude-for-financial-services

# หรือ verticals (ถ้าต้องการ skills + commands แต่ไม่เฉพาะ agent workflow)
claude plugin install investment-banking@claude-for-financial-services
claude plugin install equity-research@claude-for-financial-services
claude plugin install private-equity@claude-for-financial-services
claude plugin install fund-admin@claude-for-financial-services
```

(README.md line 59–74)

### ขั้นตอน 4: ตรวจสอบการติดตั้ง

```bash
claude plugin list
# pitch-agent@claude-for-financial-services ✓
# market-researcher@claude-for-financial-services ✓
# financial-analysis@claude-for-financial-services ✓
# ... ฯลฯ
```

### ขั้นตอน 5: ลองใช้ใน Session

สร้าง new session ด้วยคำสั่ง CLI:

```bash
claude start my-pitch-work
```

ในการสนทนา:

```
สวัสดี

ฉันต้องสร้าง pitch deck สำหรับ strategic acquisition ของ Acme Corp
 + competitor เข้ามา + 3 years strategic rationale
ฉันต้อง:
- Sector overview
- Comps analysis (8–10 peers)
- 5-year DCF
- LBO scenario
- Football field valuation
- PowerPoint deck (บน bank template)

เริ่มได้ไหม?
```

Claude จะอ่านว่า ต้อง **Pitch Agent** → เริ่ม workflow:

```
1. Confirm target, sector, situation ✓
2. Invoke sector-overview skill → draft overview
3. Query CapIQ for peers + data
4. Invoke comps-analysis skill → peer comparison
5. Invoke lbo-model, dcf-model, 3-statement-model
6. Generate football field
7. Invoke pitch-deck skill → populate PowerPoint
8. Invoke ib-check-deck → verify consistency
9. Stop & surface Excel + deck for review
```

ผลลัพธ์:
- `valuation.xlsx` (live formulas, sensitivity tables)
- `pitch_deck.pptx` (charts linked to Excel)

---

## วิธี 3: Managed Agents API (Headless) — Production

สำหรับ internal orchestration, automation, workflow engine ของตัวเอง

### Prerequisites

```bash
# 1. Export API key
export ANTHROPIC_API_KEY=sk-ant-...

# 2. Clone repo
git clone https://github.com/anthropics/claude-for-financial-services.git
cd claude-for-financial-services

# 3. Setup Python environment
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt  # includes anthropic SDK
```

### ขั้นตอน 1: Configure Environment

```bash
# Copy example env
cp .env.example .env

# Edit with your credentials
vim .env

# ตัวแปรที่จำเป็น:
# ANTHROPIC_API_KEY=sk-ant-...
# GL_MCP_URL=https://your-internal-gl.company.com/mcp      (ถ้าใช้ GL Reconciler)
# SUBLEDGER_MCP_URL=https://your-internal-subledger.com/mcp (ถ้าใช้ GL Reconciler)
# CAPIQ_API_KEY=...                                         (ถ้าใช้ CapIQ)
```

(managed-agent-cookbooks/gl-reconciler/agent.yaml line 36–37)

### ขั้นตอน 2: Deploy Agent

```bash
# Deploy GL Reconciler
scripts/deploy-managed-agent.sh gl-reconciler

# Output:
# Created agent ID: agent_ABC123...
# System prompt: ...
# Skills: gl-recon, break-tracing, variance-commentary
# MCPs: internal-gl, subledger
# Subagents: reader.yaml, critic.yaml, resolver.yaml
# Ready for /v1/agents/{id}/run
```

(README.md line 80–85)

### ขั้นตอน 3: Create Orchestration Loop

Deploy script สร้าง agent ใน Anthropic API แล้วตรง run script ลูปที่ handle `handoff_request` events:

```python
# orchestrate.py (reference implementation)
import anthropic

client = anthropic.Anthropic()
agent_id = "agent_ABC123..."  # from deploy output

# Example: GL reconciliation handoff chain
trial_balance = """
AR: $1,000,000
AP: $500,000
... 
(unbalanced by $50,000)
"""

# Send to GL Reconciler
response = client.beta.agents.runs.run(
    agent_id=agent_id,
    model="claude-opus-4-7",
    messages=[{
        "role": "user",
        "content": f"Please reconcile this trial balance:\n{trial_balance}"
    }]
)

# Poll for completion
while response.status in ["queued", "in_progress"]:
    response = client.beta.agents.runs.retrieve(agent_id, response.id)

# Handle output
if response.status == "completed":
    # Process artifacts (logs, reconciliation report)
    print(response.messages[-1].content)
elif response.status == "requires_action":
    # Route handoff_request to next subagent (reader, critic, resolver)
    # See scripts/orchestrate.py for full loop
    pass
```

(scripts/orchestrate.py — README.md line 84–85)

### ขั้นตอน 4: Route to Subagents

GL Reconciler มี depth-1 leaf workers:

```yaml
# managed-agent-cookbooks/gl-reconciler/subagents/reader.yaml
name: gl-reconciler-reader
model: claude-opus-4-7
system:
  file: ../../plugins/agent-plugins/gl-reconciler/agents/gl-reconciler.md
  append: |
    You are the Reader. Your job: ingest GL data, validate structure, identify potential breaks.
    Report findings to the critic.

callable_agents:
  - manifest: critic.yaml
```

```yaml
# managed-agent-cookbooks/gl-reconciler/subagents/critic.yaml
name: gl-reconciler-critic
system:
  append: |
    You are the Critic. Your job: analyze reader's findings, hypothesize root causes, 
    prioritize investigation paths. Route to resolver.

callable_agents:
  - manifest: resolver.yaml
```

```yaml
# managed-agent-cookbooks/gl-reconciler/subagents/resolver.yaml
name: gl-reconciler-resolver
system:
  append: |
    You are the Resolver. Your job: execute root-cause analysis, 
    draft reconciliation journal entries, route for human approval.
```

Orchestrator loop routes `handoff_request` events ตามลำดับ

---

## Verifying Installation

### Cowork: ตรวจสอบ Commands

```
Prompt: /comps Apple
# → Should invoke comps-analysis skill
# → Should query Daloopa/CapIQ MCP
# → Should return peer multiples table
```

### Claude Code CLI: ตรวจสอบ Plugins

```bash
claude plugin list | grep financial
# pitch-agent@claude-for-financial-services ✓ active
# market-researcher@claude-for-financial-services ✓ active
# financial-analysis@claude-for-financial-services ✓ active
```

### Managed Agents: ตรวจสอบ Deploy

```bash
curl -H "Authorization: Bearer $ANTHROPIC_API_KEY" \
     https://api.anthropic.com/v1/agents/agent_ABC123 | jq .status

# Should show: "status": "ready"
```

---

## First Run Examples

### Example 1: Comparative Company Analysis

```
ฉันต้องการ comps analysis สำหรับ SaaS company.
- Target: Datadog (DDOG)
- Market cap: $80B
- TTM revenue: $1.8B
- TTM EBITDA margin: 20%

ช่วยฉันหา 8-10 peers พร้อม multiples ได้ไหม?
```

**ผลลัพธ์:**
- Peer set (CloudFlare, CrowdStrike, Snyk, HashiCorp, ...)
- Trading multiples (EV/Revenue, EV/EBITDA, P/E, Price/Sales)
- Outlier flags (CrowdStrike trading at 15x vs peer median 8x — why?)
- Valuation range for Datadog

---

### Example 2: DCF Valuation

```
DCF model สำหรับ Acme Corp.
- Historical 3-year revenue CAGR: 25%
- Projected growth: 20%, 15%, 12%, 10%, 8% (next 5 years)
- EBITDA margin: 18% (expanding to 20% by year 3)
- Tax rate: 21%
- CapEx: 3% of revenue
- NWC: 5% of revenue
- Terminal growth: 3%
- WACC: 9% (Cost of Equity 11%, Cost of Debt 5%, D/V 30%)

สร้างโมเดล + sensitivity tables ได้ไหม?
```

**ผลลัพธ์:**
- Excel workbook
  - Sheet 1: Inputs (assumptions)
  - Sheet 2: Historical financials + projections
  - Sheet 3: WACC calculation
  - Sheet 4: FCF bridge → Terminal Value
  - Sheet 5: DCF valuation summary
  - Sheet 6: 5×5 sensitivity table (WACC vs Terminal g)
  - Sheet 7: Football field (comps vs precedents vs DCF vs LBO)
- All formulas live (not hardcoded)
- Sensitivity table center cell = base case price

---

## Common Customizations

### 1. Connect Your Own Data

```bash
# Edit .mcp.json to point to internal systems
vim plugins/vertical-plugins/financial-analysis/.mcp.json

# Replace MCP URLs with your own:
{
  "mcpServers": {
    "internal-gl": {
      "type": "http",
      "url": "https://gl.company.com/mcp"    # ← your GL
    },
    "internal-crm": {
      "type": "http",
      "url": "https://crm.company.com/mcp"   # ← your CRM
    },
    ...
  }
}

# Redeploy agents
scripts/deploy-managed-agent.sh gl-reconciler
```

(README.md line 154–155)

### 2. Add Firm Context to Skills

```bash
# Edit skill directly
vim plugins/vertical-plugins/financial-analysis/skills/dcf-model/SKILL.md

# Add firm guidelines:
# ...
# Your firm's WACC guidelines: Cost of Equity 10–12% (by sector), Cost of Debt 3–6%
# Use 10-year Treasury for risk-free rate
# Beta sourced from FactSet (adjusted for leverage)
# ...
```

ทั้ง Cowork + Managed Agents จะใช้ version ที่ update นี้

### 3. Create Custom Agent

```bash
# Copy existing agent
cp -r plugins/agent-plugins/pitch-agent plugins/agent-plugins/my-custom-agent

# Edit system prompt
vim plugins/agent-plugins/my-custom-agent/agents/my-custom-agent.md

# Sync skills
python3 scripts/sync-agent-skills.py

# Test + validate
python3 scripts/check.py
```

---

## ดูเพิ่มเติม

- [Architecture](/01-overview/architecture.md) — 3-layer: verticals → agents → cookbooks
- [Principles](/01-overview/principles.md) — เหตุผลที่อยู่เบื้องหลัง design
- [Full Skill Reference](https://github.com/anthropics/claude-for-financial-services#skill--command-reference) — ทั้ง commands สำหรับแต่ละ vertical
