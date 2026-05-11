---
title: KYC Screener
parent: 06 Managed Agent Cookbooks
nav_order: 7
---

# KYC Screener Cookbook

## โครงสร้าง Agent

**KYC Screener** เป็น managed agent ที่ดำเนินการ Know-Your-Customer screening อัตโนมัติ โดยตรวจสอบเอกสารลูกค้าเทียบกับหลายรายชื่อดำเนิน (sanction lists, PEP lists, AML databases)

```yaml
model: claude-opus-4-7
tools:
  - agent_toolset_20260401 (read, grep, glob เท่านั้น)
  - mcp_toolset: screening (Compliance Screening Server)
```

## Subagents

KYC Screener ประกอบด้วย 3 subagents:

| Subagent | บทบาท | หน้าที่ |
|----------|------|--------|
| `doc-reader` | Document extraction | อ่าน KYC documents (forms, IDs, PoA), extract key data |
| `rules-engine` | Compliance rules | ประเมินความเสี่ยง โดยใช้ business rules และ screening databases |
| `escalator` | Escalation & reporting | สร้าง compliance report, flag high-risk, prepare escalation |

## Compliance Workflow

KYC Screener ทำหน้าที่เป็น automated compliance officer:

1. **Document Review** — อ่านและแยกข้อมูล: name, DOB, address, beneficial ownership
2. **Screening** — เปรียบเทียบกับ OFAC, EU sanctions, PEP databases
3. **Risk Assessment** — ประเมินระดับความเสี่ยง (Low/Medium/High/Critical)
4. **Escalation** — สร้าง report, flag any issues, document ตัดสินใจ

## การ Deploy

### 1. เตรียมสภาพแวดล้อม

```bash
export SCREENING_MCP_URL="https://your-screening-mcp-server"
```

### 2. Deploy Agent

```bash
claude-code deploy ./managed-agent-cookbooks/kyc-screener/agent.yaml
```

### 3. เรียกใช้ via API

```bash
curl -X POST https://api.claude.ai/v1/agents/kyc-screener/run \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "kyc_document_path": "./documents/customer_kyc_package.pdf",
    "customer_id": "cust_789",
    "screening_level": "comprehensive",
    "jurisdiction": "EU",
    "risk_thresholds": {
      "pep_threshold": "CRITICAL",
      "sanction_threshold": "CRITICAL",
      "aml_threshold": "HIGH"
    }
  }'
```

## โครงสร้าง Output

```json
{
  "status": "completed",
  "customer_id": "cust_789",
  "screening_date": "2026-05-12",
  "customer_data": {
    "name": "John Smith",
    "date_of_birth": "1975-03-15",
    "nationality": "US",
    "address": "New York, NY",
    "id_type": "Passport",
    "id_number": "****1234"
  },
  "beneficial_ownership": {
    "direct_owner": true,
    "entities_controlled": 3,
    "high_risk_jurisdictions": 1
  },
  "screening_results": {
    "ofac_screening": {
      "status": "CLEAR",
      "matches_found": 0,
      "confidence": "100%"
    },
    "eu_sanctions": {
      "status": "CLEAR",
      "matches_found": 0
    },
    "pep_screening": {
      "status": "MATCH_FOUND",
      "matches": [
        {
          "type": "PEP - Close Associate",
          "entity": "John Smith Family Foundation",
          "connection": "Board Member",
          "risk_level": "MEDIUM"
        }
      ]
    },
    "aml_database": {
      "status": "CLEAR",
      "historical_flags": 0
    }
  },
  "risk_assessment": {
    "overall_risk_level": "MEDIUM",
    "risk_factors": [
      "PEP close association (family foundation board)",
      "Multiple entity ownership structure"
    ],
    "risk_mitigation": [
      "Enhanced due diligence recommended",
      "Obtain source of funds declaration"
    ]
  },
  "compliance_decision": {
    "recommendation": "ACCEPT_WITH_CONDITIONS",
    "conditions": [
      "Enhanced monitoring required",
      "Annual re-screening recommended",
      "Document board member status"
    ],
    "decision_maker": "REQUIRES_HUMAN_APPROVAL"
  },
  "output_files": {
    "compliance_report": "KYC_Screening_Report_cust_789.docx",
    "escalation_summary": "escalation_memo.pdf",
    "document_checklist": "document_review_checklist.xlsx"
  }
}
```

## ข้อมูล MCP Servers ที่ต้องการ

| Server | ตัวแปร Environment | ข้อมูลที่ให้บริการ |
|--------|-------------------|------------------|
| **Compliance Screening** | `SCREENING_MCP_URL` | OFAC lists, EU sanctions, PEP databases, AML watch lists |

## Structured Compliance Output

Output ของ KYC Screener ต้อง:
- Record ข้อมูล customer ที่สกัดได้
- Document screening databases ที่ใช้ + วันที่อัปเดต
- List ข้อมูล matches พร้อม confidence levels
- Provide clear recommendation: ACCEPT / REJECT / ACCEPT_WITH_CONDITIONS
- Prepare audit trail สำหรับการตรวจสอบ regulatory

## ความแตกต่างจาก Interactive Plugin

**KYC Screener Plugin** (Interactive):
- ผู้ใช้อัปโหลด KYC documents
- Plugin ถามคำถามเพิ่มเติมเกี่ยวกับวัตถุประสงค์
- ผู้ใช้ได้เห็นผลลัพธ์ step-by-step

**KYC Screener Cookbook** (Managed):
- API ให้ document path + screening parameters
- Run screening อัตโนมัติ
- ส่งออก compliance report พร้อมการตัดสินใจ

## Compliance Governance

KYC Screener output ต้อง:
- Mark clearly: **APPROVED / REJECTED / PENDING HUMAN REVIEW**
- Include timestamp ของการ screening
- Document ข้อมูล screener identity เพื่อ audit purposes
- Retain ทุกเอกสารเพื่อสอบการตรวจสอบ regulatory

---

**หมายเหตุ**: KYC Screener ต้องการ compliance screening APIs ที่ current — databases จะต้องอัปเดตทุกวัน
