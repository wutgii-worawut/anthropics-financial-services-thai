---
title: Pitch Agent
parent: 06 Managed Agent Cookbooks
nav_order: 1
---

# Pitch Agent Cookbook

## โครงสร้าง Agent

**Pitch Agent** เป็น managed agent ที่ช่วยในการสร้าง pitch decks และเอกสารการสนับสนุน โดยใช้ข้อมูล real-time

```yaml
model: claude-opus-4-7
tools:
  - agent_toolset_20260401 (read, grep, glob เท่านั้น)
  - mcp_toolset: capiq (Capital IQ)
  - mcp_toolset: daloopa (Alternative data)
```

## Subagents

Pitch Agent ประกอบด้วย 3 subagents:

| Subagent | บทบาท | หน้าที่ |
|----------|------|--------|
| `researcher` | Data gathering | ดึง market data, company fundamentals, recent news จาก Capital IQ |
| `modeler` | Financial modeling | สร้างโมเดล valuation, projections, comparable company analysis |
| `deck-writer` | Output generation | สร้าง PowerPoint deck และเอกสารสนับสนุน (มี Write permission เท่านั้น) |

## ความแตกต่างจาก Interactive Plugin

**Plugin Agent** (ทำงานใน Claude Desktop):
- Conversational, interactive
- User เห็นผลลัพธ์ทีละ step
- ใช้สำหรับการสำรวจและการวิเคราะห์ real-time

**Managed Agent** (Pitch Agent Cookbook):
- API-driven, headless
- ประมวลผลอย่างเต็มที่โดยไม่มีการโต้ตอบจากผู้ใช้
- ใช้สำหรับการสร้าง deck ที่มีโครงสร้างอย่างเต็มที่

## การ Deploy

### 1. เตรียมสภาพแวดล้อม

```bash
export CAPIQ_MCP_URL="https://your-capiq-mcp-server"
export DALOOPA_MCP_URL="https://your-daloopa-mcp-server"
```

### 2. Deploy Agent

```bash
claude-code deploy ./managed-agent-cookbooks/pitch-agent/agent.yaml
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
    "include_valuation": true
  }'
```

## โครงสร้าง Output

Pitch Agent สร้าง JSON response พร้อมเอกสารที่สร้างขึ้น:

```json
{
  "status": "completed",
  "company": "Acme Corp",
  "pitch_deck": {
    "filename": "acme_pitch_deck.pptx",
    "path": "./out/acme_pitch_deck.pptx",
    "slides": 15,
    "sections": ["cover", "investment_thesis", "market_overview", "company_profile", "valuation", "risks", "appendix"]
  },
  "supporting_documents": [
    {
      "name": "valuation_summary.xlsx",
      "sheets": ["DCF", "Comparable Companies", "Precedent Transactions"]
    },
    {
      "name": "market_research.docx",
      "pages": 8
    }
  ],
  "data_sources": {
    "capiq": "2026-05-12 snapshot",
    "daloopa": "Alternative data included"
  }
}
```

## ข้อมูล MCP Servers ที่ต้องการ

| Server | ตัวแปร Environment | ข้อมูลที่ให้บริการ |
|--------|-------------------|------------------|
| **Capital IQ** | `CAPIQ_MCP_URL` | Company data, financials, estimates, transactions |
| **Daloopa** | `DALOOPA_MCP_URL` | Alternative data, credit analytics, supply chain |

## Handoff Schema

ไม่มี handoff ระหว่าง subagents — ทั้งหมดผ่าน orchestrator (pitch-agent) เท่านั้น

---

**หมายเหตุ**: Pitch Agent ใช้ `claude-opus-4-7` สำหรับประสิทธิภาพการสร้าง deck ที่มีคุณภาพสูง
