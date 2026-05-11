---
title: Valuation Reviewer
parent: 06 Managed Agent Cookbooks
nav_order: 3
---

# Valuation Reviewer Cookbook

## โครงสร้าง Agent

**Valuation Reviewer** เป็น managed agent ที่ตรวจสอบและวิเคราะห์ valuation models เพื่อให้แน่ใจว่ามีความหมายทางเศรษฐศาสตร์ และสำเร็จลุล่วงตรรมชาติ

```yaml
model: claude-opus-4-7
tools:
  - agent_toolset_20260401 (read, grep, glob เท่านั้น)
  - mcp_toolset: portfolio (Portfolio Analytics Server)
```

## Subagents

Valuation Reviewer ประกอบด้วย 3 subagents:

| Subagent | บทบาท | หน้าที่ |
|----------|------|--------|
| `package-reader` | Package analysis | อ่านและตรวจสอบ DCF, comparables, precedent transaction packages |
| `valuation-runner` | Sensitivity analysis | รัน sensitivity analysis, scenario testing, sanity checks |
| `publisher` | Output generation | สร้าง valuation summary report พร้อมภาพและข้อเสนอแนะ |

## Headless Operation

Valuation Reviewer ทำงานแบบ fully automated:

1. รับ input: valuation package (Excel/CSV files)
2. Process: read → validate formulas → run scenarios → generate report
3. Output: validation report + updated package ใน `./out/`

## การ Deploy

### 1. เตรียมสภาพแวดล้อม

```bash
export PORTFOLIO_MCP_URL="https://your-portfolio-mcp-server"
```

### 2. Deploy Agent

```bash
claude-code deploy ./managed-agent-cookbooks/valuation-reviewer/agent.yaml
```

### 3. เรียกใช้ via API

```bash
curl -X POST https://api.claude.ai/v1/agents/valuation-reviewer/run \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "package_path": "./valuations/acme_corp_dcf.xlsx",
    "company_name": "Acme Corp",
    "review_type": "comprehensive",
    "sensitivity_factors": ["wacc", "terminal_growth", "capex_intensity"]
  }'
```

## โครงสร้าง Output

```json
{
  "status": "completed",
  "company": "Acme Corp",
  "valuation_summary": {
    "base_case_equity_value": "8.5B",
    "equity_value_per_share": "42.50",
    "valuation_range": "40B-45B per share",
    "implied_multiples": {
      "ev_ebitda": 12.3,
      "price_sales": 2.8,
      "price_fcf": 14.5
    }
  },
  "review_findings": {
    "formula_integrity": "PASSED",
    "assumptions_reasonableness": "PASSED_WITH_NOTES",
    "output_sensitivity": "MEDIUM"
  },
  "sensitivity_analysis": {
    "wacc_sensitivity": {
      "range": "7.5% - 9.5%",
      "impact": "38B-48B equity value"
    },
    "terminal_growth_sensitivity": {
      "range": "2.0% - 3.0%",
      "impact": "37B-50B equity value"
    }
  },
  "recommendations": [
    "Increase terminal growth rate assumption to 2.5% (currently 2.0%)",
    "Verify WACC calculation — current 8.2% appears slightly low vs peers"
  ],
  "output_files": {
    "validation_report": "valuation_review_report.docx",
    "updated_package": "acme_dcf_validated.xlsx",
    "sensitivity_charts": "sensitivity_analysis.pdf"
  }
}
```

## ข้อมูล MCP Servers ที่ต้องการ

| Server | ตัวแปร Environment | ข้อมูลที่ให้บริการ |
|--------|-------------------|------------------|
| **Portfolio Analytics** | `PORTFOLIO_MCP_URL` | Valuation benchmarks, peer multiples, sector norms |

## Structured Output Requirement

Output ของ Valuation Reviewer ต้องเป็น structured JSON เพื่อให้สามารถ integrate กับระบบ workflow ถัดไป:

- ข้อมูล validation (passed/failed/warning)
- ข้อมูล sensitivity ในรูปแบบ standardized
- ข้อเสนอแนะ prioritized list

## ความแตกต่างจาก Interactive Plugin

**Valuation Reviewer Plugin** (Interactive):
- ผู้ใช้เลือก DCF vs comps
- ป้อน assumptions โดยตรง
- ได้เห็นผลลัพธ์ real-time

**Valuation Reviewer Cookbook** (Managed):
- รับ package ที่สมบูรณ์
- ตรวจสอบ อัตโนมัติ
- ส่งออก validated package + report

---

**หมายเหตุ**: Valuation Reviewer ต้องการ well-formed Excel files ที่มี defined formula ranges และ input assumptions cells
