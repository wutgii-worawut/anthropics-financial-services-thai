---
title: Model Builder
parent: 06 Managed Agent Cookbooks
nav_order: 4
---

# Model Builder Cookbook

## โครงสร้าง Agent

**Model Builder** เป็น managed agent ที่สร้าง financial models อย่างเต็มตัว — ตั้งแต่ collection ข้อมูล พื้นฐาน ไปจนถึง DCF, comps, precedent transactions

```yaml
model: claude-opus-4-7
tools:
  - agent_toolset_20260401 (read, grep, glob เท่านั้น)
  - mcp_toolset: capiq (Capital IQ)
  - mcp_toolset: daloopa (Alternative data)
```

## Subagents

Model Builder ประกอบด้วย 3 subagents ที่ทำงาน sequentially:

| Subagent | บทบาท | หน้าที่ |
|----------|------|--------|
| `data-puller` | Data gathering | ดึง historical financials, guidance, peer data จาก Capital IQ |
| `builder` | Model construction | สร้าง DCF, comparables, precedent transaction schedules (มี Write) |
| `auditor` | Validation | ตรวจสอบ logic, reasonableness, consistency ของโมเดล |

## End-to-End Modeling

Model Builder ทำหน้าที่ complete modeling workflow:

1. **Data Collection** — ดึง 5 ปีของ historical data + guidance
2. **Model Build** — สร้าง 3 valuation approaches (DCF, comps, precedent)
3. **Sanity Check** — ตรวจสอบ formulas, assumptions, output consistency
4. **Package** — ส่งออก Excel workbook ที่ ready-to-use

## การ Deploy

### 1. เตรียมสภาพแวดล้อม

```bash
export CAPIQ_MCP_URL="https://your-capiq-mcp-server"
export DALOOPA_MCP_URL="https://your-daloopa-mcp-server"
```

### 2. Deploy Agent

```bash
claude-code deploy ./managed-agent-cookbooks/model-builder/agent.yaml
```

### 3. เรียกใช้ via API

```bash
curl -X POST https://api.claude.ai/v1/agents/model-builder/run \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "company_ticker": "MSFT",
    "company_name": "Microsoft Corporation",
    "modeling_period": 10,
    "include_dcf": true,
    "include_comps": true,
    "include_precedent": true,
    "terminal_growth_rate": 0.025
  }'
```

## โครงสร้าง Output

```json
{
  "status": "completed",
  "company": "Microsoft Corporation",
  "ticker": "MSFT",
  "model_package": {
    "filename": "MSFT_Financial_Model_2026.xlsx",
    "sheets": [
      "Historical",
      "Assumptions",
      "DCF_Analysis",
      "Comparables",
      "Precedent_Transactions",
      "Sensitivity",
      "Summary"
    ]
  },
  "data_sourcing": {
    "source": "Capital IQ",
    "period": "2021-2025 historical + 2026-2030 guidance",
    "completeness": "100%",
    "last_updated": "2026-05-12"
  },
  "dcf_analysis": {
    "wacc": "7.8%",
    "terminal_growth": "2.5%",
    "equity_value": "2.85T",
    "value_per_share": "380"
  },
  "comps_analysis": {
    "peer_group": 8,
    "median_ev_ebitda": 24.5,
    "median_price_sales": 8.2,
    "implied_equity_value": "2.92T",
    "implied_value_per_share": "388"
  },
  "precedent_analysis": {
    "comparable_transactions": 5,
    "median_transaction_multiple": 23.1,
    "implied_value_estimate": "2.78T",
    "implied_value_per_share": "370"
  },
  "valuation_summary": {
    "dcf_value": 380,
    "comps_value": 388,
    "precedent_value": 370,
    "blended_value": 379,
    "valuation_range": "370-388"
  },
  "audit_results": {
    "formula_check": "PASSED",
    "assumption_reasonableness": "PASSED",
    "cross_sheet_consistency": "PASSED",
    "issues": []
  }
}
```

## ข้อมูล MCP Servers ที่ต้องการ

| Server | ตัวแปร Environment | ข้อมูลที่ให้บริการ |
|--------|-------------------|------------------|
| **Capital IQ** | `CAPIQ_MCP_URL` | Financials, guidance, peer data, transaction history |
| **Daloopa** | `DALOOPA_MCP_URL` | Alternative data, real-time metrics, supply chain insights |

## Model Architecture

Model Builder สร้าง Excel workbook ที่มี:

- **Historical Sheet** — 5 ปีของ income statement, balance sheet, cash flow
- **Assumptions Sheet** — WACC, growth rates, margins (user-editable)
- **DCF Sheet** — Projection + NPV calculation
- **Comps Sheet** — Peer multiples + implied values
- **Precedent Sheet** — Transaction multiples + implied values
- **Sensitivity Sheet** — 2-way sensitivity tables
- **Summary Sheet** — Valuation summary + waterfall

## ความแตกต่างจาก Interactive Plugin

**Model Builder Plugin** (Interactive):
- ผู้ใช้เลือก company
- Plugin ถามคำถามเพิ่มเติมเกี่ยวกับ assumptions
- ผู้ใช้ได้เห็น model build ขั้นต่อ ขั้นต่อ

**Model Builder Cookbook** (Managed):
- API ให้ company + key parameters
- Create complete model อัตโนมัติ
- ส่งออก ready-to-use Excel file

---

**หมายเหตุ**: Model Builder ใช้ Claude Opus 4.7 สำหรับความแม่นยำสูง ในการตั้งค่า formulas และ logic
