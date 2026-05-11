---
title: Earnings Reviewer
parent: 06 Managed Agent Cookbooks
nav_order: 2
---

# Earnings Reviewer Cookbook

## โครงสร้าง Agent

**Earnings Reviewer** เป็น managed agent ที่อ่านและวิเคราะห์ earnings call transcripts จากนั้นอัปเดตโมเดลทางการเงินโดยอัตโนมัติ

```yaml
model: claude-opus-4-7
tools:
  - agent_toolset_20260401 (read, grep, glob เท่านั้น)
  - mcp_toolset: factset (FactSet data)
  - mcp_toolset: daloopa (Alternative data)
```

## Subagents

Earnings Reviewer ประกอบด้วย 3 subagents:

| Subagent | บทบาท | หน้าที่ |
|----------|------|--------|
| `transcript-reader` | Transcript analysis | อ่าน earnings call transcript, extract guidance, color, tone |
| `model-updater` | Financial modeling | อัปเดต Excel model จากข้อมูล guidance ใหม่ |
| `note-writer` | Documentation | เขียน analyst note และเอกสารสรุป (มี Write permission เท่านั้น) |

## Headless Operation

Earnings Reviewer ทำงาน **headless** — ไม่มี interactive input ระหว่าง process:

1. รับ input: earnings call transcript + current Excel model
2. Process: analyze → model update → documentation
3. Output: updated model + analyst note ใน `./out/`

## การ Deploy

### 1. เตรียมสภาพแวดล้อม

```bash
export FACTSET_MCP_URL="https://your-factset-mcp-server"
export DALOOPA_MCP_URL="https://your-daloopa-mcp-server"
```

### 2. Deploy Agent

```bash
claude-code deploy ./managed-agent-cookbooks/earnings-reviewer/agent.yaml
```

### 3. เรียกใช้ via API

```bash
curl -X POST https://api.claude.ai/v1/agents/earnings-reviewer/run \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "company_ticker": "AAPL",
    "transcript_path": "./transcripts/aapl_q2_2026_earnings.txt",
    "current_model_path": "./models/aapl_model.xlsx",
    "analyst_name": "Research Team"
  }'
```

## โครงสร้าง Output

```json
{
  "status": "completed",
  "company": "Apple Inc",
  "ticker": "AAPL",
  "earnings_date": "2026-05-10",
  "transcript_analysis": {
    "guidance_fy2026": {
      "revenue": "95B-98B",
      "gross_margin": "43%-44%",
      "eps": "7.50-7.75"
    },
    "key_themes": ["services growth", "geographic weakness", "capex investment"],
    "tone": "cautious optimism"
  },
  "model_updates": {
    "filename": "aapl_model_updated.xlsx",
    "changes": ["FY2026 revenue revised down 2%", "Services upside adjusted", "Capex increased 8%"],
    "impact_on_valuation": "-3.2% DCF"
  },
  "analyst_note": {
    "filename": "aapl_earnings_note.docx",
    "pages": 4,
    "sections": ["summary", "key_takeaways", "model_changes", "view_update"]
  }
}
```

## ข้อมูล MCP Servers ที่ต้องการ

| Server | ตัวแปร Environment | ข้อมูลที่ให้บริการ |
|--------|-------------------|------------------|
| **FactSet** | `FACTSET_MCP_URL` | Historical earnings, consensus, guidance history |
| **Daloopa** | `DALOOPA_MCP_URL` | Real-time market reaction, sentiment data |

## ความแตกต่างจาก Plugin Agent

**Earnings Reviewer Plugin** (Interactive):
- ผู้ใช้อัปโหลด transcript
- Agent ถามคำถามเพิ่มเติม
- ผู้ใช้เห็นผลลัพธ์ step-by-step

**Earnings Reviewer Cookbook** (Managed):
- Fully automated, API-triggered
- ไม่มี user interaction
- Output files ไปที่ `./out/` โดยอัตโนมัติ

---

**หมายเหตุ**: Model update ใช้ Excel API — โมเดลต้องเป็น structured spreadsheet ที่มี defined ranges สำหรับ input
