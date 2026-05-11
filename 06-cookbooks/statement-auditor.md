---
title: Statement Auditor
parent: 06 Managed Agent Cookbooks
nav_order: 8
---

# Statement Auditor Cookbook

## โครงสร้าง Agent

**Statement Auditor** เป็น managed agent ที่ตรวจสอบ fund statements, NAV statements, และเอกสารการกำหนดราคา โดยตรวจหาข้อผิดพลาด inconsistencies และการแยกส่วนที่น่าสงสัย

```yaml
model: claude-opus-4-7
tools:
  - agent_toolset_20260401 (read, grep, glob เท่านั้น)
  - mcp_toolset: nav (NAV Calculation & Pricing Server)
```

## Subagents

Statement Auditor ประกอบด้วย 3 subagents:

| Subagent | บทบาท | หน้าที่ |
|----------|------|--------|
| `statement-reader` | Document parsing | อ่าน fund statements, extract holdings, NAV, pricing data |
| `reconciler` | Reconciliation | ตรวจสอบ: NAV calculations vs pricing server, prior period rolls forward |
| `flagger` | Anomaly detection | ระบุ outliers, unexplained changes, data quality issues |

## NAV Audit Workflow

Statement Auditor ทำหน้าที่เป็น automated NAV auditor:

1. **Statement Review** — อ่าน statement: holdings, NAV, fee detail
2. **Recalculation** — Recalculate NAV using pricing server เพื่อ verify
3. **Reconciliation** — Compare กับ prior month, identify exceptions
4. **Flagging** — List any issues, unexplained gaps, data anomalies
5. **Report** — Generate audit report พร้อมข้อเสนอแนะ

## การ Deploy

### 1. เตรียมสภาพแวดล้อม

```bash
export NAV_MCP_URL="https://your-nav-pricing-mcp-server"
```

### 2. Deploy Agent

```bash
claude-code deploy ./managed-agent-cookbooks/statement-auditor/agent.yaml
```

### 3. เรียกใช้ via API

```bash
curl -X POST https://api.claude.ai/v1/agents/statement-auditor/run \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "statement_path": "./documents/fund_nav_statement_2026_05.pdf",
    "fund_id": "fund_123",
    "statement_date": "2026-05-31",
    "prior_statement_date": "2026-04-30",
    "audit_level": "comprehensive"
  }'
```

## โครงสร้าง Output

```json
{
  "status": "completed",
  "fund_id": "fund_123",
  "statement_date": "2026-05-31",
  "statement_summary": {
    "nav_per_share": "105.32",
    "total_nav": "525.6M",
    "number_of_shares": "4,987,250",
    "number_of_holdings": 47
  },
  "nav_verification": {
    "stated_nav": "525.6M",
    "recalculated_nav": "525.58M",
    "difference": "0.02M",
    "difference_pct": "0.004%",
    "verification_status": "PASS"
  },
  "reconciliation": {
    "prior_nav": "518.3M",
    "inflows": "8.5M",
    "outflows": "-2.1M",
    "investment_gains": "4.8M",
    "fees": "-3.4M",
    "reconciled_nav": "525.6M",
    "reconciliation_status": "PASS"
  },
  "holding_audit": {
    "total_holdings_counted": 47,
    "pricing_source_errors": 0,
    "stale_price_flags": [
      {
        "holding": "ACME Corp Bond",
        "isin": "US0123456789",
        "last_price_date": "2026-05-25",
        "days_stale": 6,
        "price_impact": "12k on 5.2M position"
      }
    ]
  },
  "exceptions_and_flags": {
    "high_priority": [
      {
        "flag": "Leverage ratio exceeded",
        "value": "35.2%",
        "limit": "30%",
        "action_required": "true"
      }
    ],
    "medium_priority": [
      {
        "flag": "One holding >5% decrease",
        "holding": "XYZ Fund",
        "change": "-8.2%",
        "dollar_impact": "-21.5M",
        "explanation": "Liquidation for redemptions"
      }
    ],
    "low_priority": [
      {
        "flag": "Rounding difference",
        "amount": "0.01k",
        "status": "Acceptable"
      }
    ]
  },
  "audit_summary": {
    "overall_status": "FLAG_FOR_REVIEW",
    "critical_issues": 1,
    "items_requiring_explanation": 2,
    "quality_assessment": "GOOD"
  },
  "recommendations": [
    "Obtain explanation for leverage ratio breach",
    "Update stale pricing on ACME bond",
    "Document XYZ Fund liquidation rationale"
  ],
  "output_files": {
    "audit_report": "NAV_Audit_Report_2026_05_31.docx",
    "exception_detail": "exceptions_detail.xlsx",
    "reconciliation_workpaper": "reconciliation_wp.xlsx"
  }
}
```

## ข้อมูล MCP Servers ที่ต้องการ

| Server | ตัวแปร Environment | ข้อมูลที่ให้บริการ |
|--------|-------------------|------------------|
| **NAV/Pricing** | `NAV_MCP_URL` | Pricing lookups, currency conversions, NAV calculation templates |

## Audit Findings Classification

Statement Auditor จัดประเภท findings:

- **Critical** — Breaches of investment policy, leverage limits, regulatory requirements
- **High** — Data integrity issues, large unexplained changes
- **Medium** — Explanable outliers, stale data, minor calculation gaps
- **Low** — Rounding, immaterial discrepancies

## Structured Output Format

Output ต้อง:
- Provide reconciliation detail (แสดงขั้นตอนการคำนวณ)
- List ข้อ exceptions พร้อม severity + dollar impact
- Document pricing sources and dates
- Prepare audit trail สำหรับ fund administrator review

## ความแตกต่างจาก Interactive Plugin

**Statement Auditor Plugin** (Interactive):
- ผู้ใช้อัปโหลด statement
- Plugin ถามคำถามเกี่ยวกับ prior month data
- ผู้ใช้ได้เห็นผลลัพธ์ real-time

**Statement Auditor Cookbook** (Managed):
- API ให้ statement path + fund ID
- Run audit อัตโนมัติ
- ส่งออก audit report พร้อมเอกสารประกอบ

## Compliance & Control Governance

Statement Auditor output ต้อง:
- Document audit date, auditor identity (traceability)
- Support audit trail สำหรับ fund administrator sign-off
- Flag ข้อ exceptions that require explanation
- Prepare recommendation สำหรับ remediation

---

**หมายเหตุ**: Statement Auditor มีประโยชน์สำหรับ month-end close process — สามารถรัน parallel with manual NAV process
