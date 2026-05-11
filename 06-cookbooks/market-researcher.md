---
title: Market Researcher
parent: 06 Managed Agent Cookbooks
nav_order: 5
---

# Market Researcher Cookbook

## โครงสร้าง Agent

**Market Researcher** เป็น managed agent ที่วิเคราะห์ภาคส่วนและสร้าง market research reports โดยใช้ข้อมูลจาก Capital IQ และ FactSet

```yaml
model: claude-opus-4-7
tools:
  - agent_toolset_20260401 (read, grep, glob เท่านั้น)
  - mcp_toolset: capiq (Capital IQ)
  - mcp_toolset: factset (FactSet)
```

## Subagents

Market Researcher ประกอบด้วย 3 subagents:

| Subagent | บทบาท | หน้าที่ |
|----------|------|--------|
| `sector-reader` | Sector analysis | อ่าน sector trends, industry dynamics, regulatory changes |
| `comps-spreader` | Comparable analysis | สร้าง peer comparison matrices, multiples analysis, relative value |
| `note-writer` | Report generation | เขียน market research note พร้อมสรุป insights และ implications |

## Market Research Workflow

Market Researcher ทำหน้าที่เป็น comprehensive sector analyst:

1. **Sector Overview** — ตรวจสอบ industry tailwinds/headwinds, growth trends
2. **Peer Comparison** — ดึง financials ของบริษัทในภาคส่วน สร้าง comparison table
3. **Valuation Context** — วิเคราะห์ relative valuation, trading ranges
4. **Report** — เขียน market research note พร้อมข้อเสนอแนะ

## การ Deploy

### 1. เตรียมสภาพแวดล้อม

```bash
export CAPIQ_MCP_URL="https://your-capiq-mcp-server"
export FACTSET_MCP_URL="https://your-factset-mcp-server"
```

### 2. Deploy Agent

```bash
claude-code deploy ./managed-agent-cookbooks/market-researcher/agent.yaml
```

### 3. เรียกใช้ via API

```bash
curl -X POST https://api.claude.ai/v1/agents/market-researcher/run \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "sector": "Technology",
    "subsector": "Cloud Infrastructure",
    "focus_countries": ["US", "EU"],
    "include_valuation_analysis": true,
    "analyst_perspective": "Buy-side"
  }'
```

## โครงสร้าง Output

```json
{
  "status": "completed",
  "sector": "Technology - Cloud Infrastructure",
  "report_date": "2026-05-12",
  "sector_overview": {
    "market_size": "850B globally",
    "growth_rate": "18% CAGR",
    "key_drivers": ["AI adoption", "hybrid cloud migration", "edge computing"],
    "headwinds": ["vendor consolidation", "pricing pressure"]
  },
  "peer_group": {
    "count": 12,
    "companies": [
      "Amazon (AWS)",
      "Microsoft (Azure)",
      "Google (GCP)",
      "Alibaba",
      "IBM",
      "Oracle"
    ]
  },
  "comparative_metrics": {
    "data": [
      {
        "company": "Amazon",
        "revenue_growth": "15.2%",
        "ebitda_margin": "28.5%",
        "ev_revenue": 8.2,
        "ev_ebitda": 28.7
      },
      {
        "company": "Microsoft",
        "revenue_growth": "16.8%",
        "ebitda_margin": "32.1%",
        "ev_revenue": 12.5,
        "ev_ebitda": 38.9
      }
    ],
    "sector_medians": {
      "revenue_growth": "14.5%",
      "ebitda_margin": "22.3%",
      "ev_revenue": 9.1,
      "ev_ebitda": 40.2
    }
  },
  "valuation_insights": {
    "assessment": "Sector trading at 15% premium to historical",
    "market_leaders": "Commanding 20-30% valuation premium",
    "pure_plays": "Trading at sector discount"
  },
  "investment_themes": [
    "AI enablement driving cloud adoption",
    "Margin expansion opportunities",
    "M&A consolidation likely to continue"
  ],
  "output_files": {
    "market_research_report": "cloud_sector_research_2026_05.docx",
    "peer_comparison": "cloud_peer_matrix.xlsx",
    "valuation_summary": "valuation_summary.pdf"
  }
}
```

## ข้อมูล MCP Servers ที่ต้องการ

| Server | ตัวแปร Environment | ข้อมูลที่ให้บริการ |
|--------|-------------------|------------------|
| **Capital IQ** | `CAPIQ_MCP_URL` | Company financials, sector classifications, peer lists |
| **FactSet** | `FACTSET_MCP_URL` | Industry metrics, sentiment data, analyst reports metadata |

## Structured Output Format

Output ของ Market Researcher ต้องเป็น structured JSON ที่สามารถ:
- Export to Excel peer matrix
- Generate PDF summary
- Feed into pitch documents or investment committee memos

## ความแตกต่างจาก Interactive Plugin

**Market Researcher Plugin** (Interactive):
- ผู้ใช้ระบุ sector of interest
- Plugin ให้ suggestions ขณะทำการวิจัย
- ผู้ใช้สามารถ drill down แต่ละเรื่อง

**Market Researcher Cookbook** (Managed):
- API ให้ sector + focus areas
- Complete research อัตโนมัติ
- ส่งออก finished report + data files

---

**หมายเหตุ**: Market Researcher มีประโยชน์สำหรับ sector rotation analysis หรือ pitch preparation สำหรับ fund strategy
