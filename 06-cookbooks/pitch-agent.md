---
title: Pitch Agent
parent: 06 Managed Agent Cookbooks
nav_order: 1
---

# Pitch Agent Cookbook

## โครงสร้าง Agent

**Pitch Agent** เป็น managed agent ที่ช่วยในการสร้าง pitch decks และเอกสารการสนับสนุน โดยใช้ข้อมูล real-time จากแหล่งข้อมูลทางการเงิน

Agent รันแบบ **headless** (ไม่มี interactive UI) เพื่อให้เหมาะสำหรับการใช้งานผ่าน API อัตโนมัติ ทั้งจากระบบ orchestration ของบริษัทหรือจากแอปพลิเคชันอื่น

```yaml
# จาก agent.yaml (บรรทัด 3-18)
name: pitch-agent
model: claude-opus-4-7

system:
  file: ../../plugins/agent-plugins/pitch-agent/agents/pitch-agent.md
  append: "You are running headless. Produce files in ./out/; do not assume an open Office document."

tools:
  - type: agent_toolset_20260401
    default_config: { enabled: false }
    configs:
      - { name: read,  enabled: true }
      - { name: grep,  enabled: true }
      - { name: glob,  enabled: true }
  - { type: mcp_toolset, mcp_server_name: capiq,   default_config: { enabled: true } }
  - { type: mcp_toolset, mcp_server_name: daloopa, default_config: { enabled: true } }
```

## Subagents และบทบาทของแต่ละตัว

Pitch Agent ประกอบด้วย 3 subagents ที่มีการแบ่งหน้าที่ชัดเจนในการสร้าง pitch:

| Subagent | Model | บทบาท | Tools | MCP Connectors |
|----------|-------|--------|-------|---|
| `researcher` | claude-opus-4-7 | Data gathering & comps analysis | read, grep | CapIQ, Daloopa (read-only) |
| `modeler` | claude-opus-4-7 | Financial modeling | read, bash (sandboxed) | CapIQ, Daloopa (read-only) |
| `deck-writer` | claude-opus-4-7 | Document generation | read, write, edit | None |

### Subagent: Researcher

**หน้าที่**: ดึงข้อมูลทางการเงินจากแหล่งข้อมูล Capital IQ และ Daloopa
- ดึง market data และ company fundamentals
- ทำ comparable company analysis (trading multiples, precedent transactions)
- คืนผล structured JSON ที่มี schema validation (output_schema บรรทัด 21-47 ของ researcher.yaml):
  ```
  {
    "target": "Target Company",
    "comps": [
      { "ticker": "COMP1", "metric": "EV/Revenue", "value": 8.5 },
      { "ticker": "COMP1", "metric": "P/E", "value": 22.3 }
    ],
    "precedents": [
      { "target": "Acme", "acquirer": "MegaCorp", "ev": 12000000000, "multiple": 9.5 }
    ]
  }
  ```

### Subagent: Modeler

**หน้าที่**: สร้างโมเดลทางการเงินจากข้อมูลที่ researcher ดึงมา
- สร้าง DCF valuation models
- สร้าง comparable company valuation analysis
- สร้าง precedent transaction analysis
- คำนวณ implied valuations สำหรับ pitch thesis

### Subagent: Deck-Writer (Write Holder)

**หน้าที่**: สร้างเอกสารสุดท้าย — PowerPoint deck และ Excel supporting files
- โปรแกรม PowerPoint ใช้ `pptx-author` helper function
- โปรแกรม Excel ใช้ `xlsx-author` helper function
- ไม่มี access ไปที่ MCP servers — รับข้อมูลจาก modeler เท่านั้น
- **เป็น worker เพียงตัวเดียวที่มี Write permission** สำหรับ file system

## ขั้นตอนการทำงานภายใน (Internal Workflow)

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Pitch Agent Orchestrator (Main)                         │
│    - Receives: { company_name, pitch_type, ... }           │
│    - Sets context from request                             │
└────────────────────┬────────────────────────────────────────┘
                     │
        ┌────────────┴────────────┬──────────────────┐
        ▼                         ▼                  ▼
   ┌─────────┐         ┌──────────────┐      ┌──────────────┐
   │researcher│        │   modeler    │      │ deck-writer  │
   │          │        │              │      │              │
   │ Query    │        │ Take comps & │      │ Format data  │
   │ CapIQ/  │───────▶│ precedents;  │─────▶│ into PPTX    │
   │ Daloopa │        │ build models │      │ and XLSX     │
   │          │        │              │      │              │
   └─────────┘        └──────────────┘      └──────────────┘
        │                    │                     │
        │                    │                     │
        └────────────────────┼─────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  ./out/
                    │  ├── pitch-<target>.pptx
                    │  ├── model.xlsx
                    │  └── support docs
                    └─────────────────┘
```

## Skills ที่ Agent เรียกใช้

Pitch Agent loads skills จาก plugin bundle `pitch-agent`:

```yaml
skills:
  - { from_plugin: ../../plugins/agent-plugins/pitch-agent }
```

Skills อาจรวมถึง:
- `pull-company-data` — Query company profile, financials, news
- `comparable-company-analysis` — Build comps table
- `precedent-analysis` — M&A transaction benchmarking
- `dcf-valuation` — DCF modeling helper
- `thesis-formatting` — Convert analysis into pitch narrative

(ชื่อ skill จริงขึ้นอยู่กับ source plugin)

## MCP Servers & Environment Setup

### 1. Capital IQ (CapIQ) MCP Server

```bash
export CAPIQ_MCP_URL="https://your-capiq-mcp-server"
```

ทำให้ available ผ่าน tools ด้านล่าง:
- `get_company_data(ticker/name)` — Company profile, financials
- `get_market_data()` — Stock prices, market cap, sector data
- `get_comparable_companies()` — Peer identification & multiples
- `get_precedent_transactions()` — M&A activity

### 2. Daloopa MCP Server

```bash
export DALOOPA_MCP_URL="https://your-daloopa-mcp-server"
```

ทำให้ available ผ่าน tools:
- `get_alternative_data()` — Credit analytics, supply chain intelligence
- `get_sentiment()` — Real-time news sentiment
- `get_supply_chain_data()` — Key relationships, risk exposure

## การ Deploy

### 1. เตรียมสภาพแวดล้อม

```bash
# Set environment variables for MCP servers
export ANTHROPIC_API_KEY=sk-ant-...
export CAPIQ_MCP_URL="https://your-capiq-server/mcp"
export DALOOPA_MCP_URL="https://your-daloopa-server/mcp"

# Verify MCP server URLs are reachable
curl -s $CAPIQ_MCP_URL/health | jq .
curl -s $DALOOPA_MCP_URL/health | jq .
```

### 2. Deploy Agent

```bash
cd /path/to/financial-services
../../scripts/deploy-managed-agent.sh pitch-agent

# Output:
# ✓ Validating manifest: managed-agent-cookbooks/pitch-agent/agent.yaml
# ✓ Loading skills from agent-plugins/pitch-agent
# ✓ Deploying to Claude Agents API
# Agent deployed: pitch-agent (ID: ag-xyz...)
```

### 3. เรียกใช้ via API

```bash
curl -X POST https://api.claude.ai/v1/agents/pitch-agent/run \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "company_name": "Acme Corp",
    "pitch_type": "equity_research",
    "include_comps": true,
    "include_valuation": true,
    "target_audience": "buy-side",
    "output_path": "./out"
  }'
```

## โครงสร้าง Output (Response)

```json
{
  "status": "completed",
  "company": "Acme Corp",
  "ticker": "ACE",
  "execution_time_ms": 45000,
  "pitch_deck": {
    "filename": "pitch-acme-corp-20260512.pptx",
    "path": "./out/pitch-acme-corp-20260512.pptx",
    "slides": 15,
    "sections": [
      "cover",
      "executive_summary",
      "investment_thesis",
      "market_overview",
      "company_profile",
      "financial_analysis",
      "valuation_summary",
      "comparable_companies",
      "precedent_transactions",
      "risks",
      "appendix"
    ]
  },
  "supporting_documents": [
    {
      "name": "valuation_models.xlsx",
      "size_bytes": 156000,
      "sheets": [
        "DCF Model",
        "Comparable Companies",
        "Precedent Transactions",
        "Summary Metrics"
      ]
    },
    {
      "name": "market_research.docx",
      "size_bytes": 89000,
      "pages": 8
    }
  ],
  "data_sources": {
    "capiq_snapshot_date": "2026-05-12T14:32:00Z",
    "daloopa_snapshot_date": "2026-05-12T14:30:00Z",
    "financials_period": "Q1 2026"
  },
  "metrics_summary": {
    "comparable_companies": 7,
    "precedent_transactions": 12,
    "valuation_range": {
      "low": 8.2,
      "base": 9.5,
      "high": 11.3
    }
  }
}
```

## Edge Cases และ Failure Modes

### 1. Missing or Incomplete MCP Data

**ปัญหา**: CapIQ ไม่ return comps เพราะบริษัท target เป็น niche player

**การ handle**:
- Researcher ใช้ `get_competitors_from_identifiers` แบบ broad
- Manual competitor identification จาก industry knowledge
- Document missing data points ในเอกสาร supporting

### 2. Stale Financial Data

**ปัญหา**: Target company ไม่เคยมี quarterly filings ล่าสุด

**การ handle**:
- Agent เตือน: "Using most recent available data: [period]"
- LTM metrics คำนวณจากข้อมูลที่มี (partial year + prior years)
- ระบุ data lag ใน footer ของ pitch deck

### 3. Model Convergence Issues

**ปัญหา**: DCF model sensitivity analysis แสดง wide valuation range (e.g., $5–$20/share)

**การ handle**:
- Modeler ปรับ assumption ranges เพื่อidentify key drivers
- Deck-writer highlight "high sensitivity to revenue growth assumptions"
- เพิ่ม scenario analysis sheet ใน Excel

### 4. File Write Permission Errors

**ปัญหา**: `deck-writer` agent ไม่สามารถ write ไปที่ `./out/` directory

**การ handle**:
- Check directory permissions: `ls -ld ./out/`
- Create if missing: `mkdir -p ./out`
- Check disk space: `df -h .`

## การ Debug

### 1. Enable Logging

```bash
export CLAUDE_LOG_LEVEL=DEBUG
../../scripts/deploy-managed-agent.sh pitch-agent
```

ตรวจสอบ logs:
```bash
tail -f ./pitch-agent.log
# [researcher] Calling get_company_data(ticker='ACE')
# [researcher] Received 42 competitors
# [modeler] Building DCF model...
# [deck-writer] Creating PowerPoint with 15 slides
```

### 2. Test Individual Subagents

```bash
# Test researcher subagent only
curl -X POST https://api.claude.ai/v1/agents/pitch-researcher/run \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"company_name": "Apple Inc"}'

# Should return comps and precedents only (no deck)
```

### 3. Inspect Intermediate Files

```bash
# If agent supports debug output
cat ./out/researcher-output.json | jq .comps | head -20
cat ./out/modeler-output.json | jq .dcf_summary

# Verify deck was created
file ./out/pitch-*.pptx
unzip -l ./out/pitch-*.pptx | grep -c "slide"
```

## ตัวอย่าง Output แบบเต็ม

### Request

```json
{
  "company_name": "Stripe Inc",
  "pitch_type": "growth_equity",
  "include_comps": true,
  "include_valuation": true,
  "target_audience": "growth_investors",
  "comparable_count": 8,
  "precedent_years": 5
}
```

### Response (Abbreviated)

```json
{
  "status": "completed",
  "company": "Stripe Inc",
  "execution_time_ms": 52000,
  "pitch_deck": {
    "filename": "pitch-stripe-inc-20260512.pptx",
    "slides": 18,
    "sections": [
      "cover: Stripe — Payments for the Internet",
      "executive_summary: 5 key investment points",
      "market_overview: $10T+ payments TAM",
      "company_profile: Global payments platform",
      "financial_analysis: Revenue $5.2B (2024), 24% CAGR",
      "valuation_summary: Base case $95B, Range $75–$115B",
      "comparable_companies: Visa, Mastercard, FIS, etc.",
      "precedent_transactions: Recent fintech M&A"
    ]
  },
  "supporting_documents": [
    {
      "name": "stripe_valuation_models.xlsx",
      "sheets": [
        "DCF Model (3-year, 5-year, terminal)",
        "Trading Comps (P/E, EV/Revenue, etc.)",
        "Precedent Transactions (recent deals)",
        "Summary Metrics (IRR, NPV, sensitivity)"
      ]
    }
  ],
  "metrics_summary": {
    "valuation_base_case_usd_billions": 95,
    "valuation_low_usd_billions": 75,
    "valuation_high_usd_billions": 115,
    "implied_revenue_multiple_2024": 18.3,
    "comparable_companies_used": 8,
    "precedent_transactions_analyzed": 12
  }
}
```

## ความแตกต่างจาก Interactive Plugin

**Plugin Agent** (ทำงานใน Claude Desktop):
- Conversational, interactive — user เห็นผลลัพธ์ทีละ step
- User ถาม follow-up questions
- Real-time adjustments to pitch thesis
- ใช้สำหรับการสำรวจและการวิเคราะห์

**Managed Agent** (Pitch Agent Cookbook):
- API-driven, headless — ไม่มี user interaction
- ประมวลผลอย่างเต็มที่ใน single run
- ใช้สำหรับ batch processing (ตัวอย่าง: generate deck สำหรับ 50 targets)
- Output files ไปยัง directory โดยอัตโนมัติ

## Handoff Schema & Orchestration

**หมายเหตุสำคัญ**: ไม่มี handoff ระหว่าง subagents ในแบบ traditional sense — ทั้งหมดผ่าน orchestrator เท่านั้น

```yaml
callable_agents:
  - { manifest: ./subagents/researcher.yaml }    # step 1
  - { manifest: ./subagents/modeler.yaml }       # step 2 (input: researcher output)
  - { manifest: ./subagents/deck-writer.yaml }   # step 3 (input: modeler output)
```

**Orchestration Flow:**
```
API Request
    ↓
Pitch Agent (orchestrator)
    ├─→ Call researcher subagent
    │   └─→ Returns: JSON with comps, precedents
    │
    ├─→ Call modeler subagent (with researcher output)
    │   └─→ Returns: JSON with valuations, models
    │
    └─→ Call deck-writer subagent (with all prior outputs)
        └─→ Returns: { status: "completed", files: [...] }
         └─→ Writes to ./out/
```

---

**หมายเหตุ**: Pitch Agent ใช้ `claude-opus-4-7` สำหรับประสิทธิภาพการสร้าง deck ที่มีคุณภาพสูง (complex valuation logic + document generation)
