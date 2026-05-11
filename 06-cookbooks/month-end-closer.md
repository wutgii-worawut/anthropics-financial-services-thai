---
title: Month-End Closer
parent: 06 Managed Agent Cookbooks
nav_order: 10
---

# Month-End Closer Cookbook

## โครงสร้าง Agent

**Month-End Closer** เป็น managed agent ที่ดำเนินการ month-end close process โดยอัตโนมัติ — เตรียมรายการประเมินค่า, accruals, และรายการสะท้อนกลับ (reversal entries)

```yaml
model: claude-opus-4-7
tools:
  - agent_toolset_20260401 (read, grep, glob เท่านั้น)
  - mcp_toolset: internal-gl (Internal GL Server)
```

## Subagents

Month-End Closer ประกอบด้วย 3 subagents:

| Subagent | บทบาท | หน้าที่ |
|----------|------|--------|
| `ledger-reader` | Transaction extraction | อ่าน GL transactions สำหรับเดือนปิด, identify potential accruals |
| `rollforward` | Month-end calculations | Calculate accruals (utility, interest, payroll), prepare reversal entries |
| `poster` | Entry creation | สร้าง month-end journal entries, prepare posting list (มี Write permission เท่านั้น) |

## Month-End Close Workflow

Month-End Closer ทำหน้าที่เป็น automated close coordinator:

1. **Transaction Review** — อ่าน GL activity สำหรับ period, identify missing accruals
2. **Accrual Calculation** — Calculate: utility accruals, interest expense, payroll payable, etc.
3. **Reversal Preparation** — สร้าง reversals สำหรับ prior month accruals
4. **Entry Posting** — Post month-end JE ไปยัง GL, prepare close checklist

## การ Deploy

### 1. เตรียมสภาพแวดล้อม

```bash
export GL_MCP_URL="https://your-internal-gl-server"
```

### 2. Deploy Agent

```bash
claude-code deploy ./managed-agent-cookbooks/month-end-closer/agent.yaml
```

### 3. เรียกใช้ via API

```bash
curl -X POST https://api.claude.ai/v1/agents/month-end-closer/run \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "period": "2026-05",
    "close_date": "2026-05-31",
    "accrual_targets": ["utilities", "interest_expense", "payroll_payable"],
    "prior_period_reversals": true,
    "posting_sequence": "standard"
  }'
```

## โครงสร้าง Output

```json
{
  "status": "completed",
  "period": "2026-05",
  "close_date": "2026-05-31",
  "close_summary": {
    "transactions_analyzed": 2847,
    "accruals_identified": 8,
    "entries_to_post": 12,
    "gl_accounts_affected": 34
  },
  "accrual_detail": {
    "utilities_accrual": {
      "estimate": "45,300",
      "calculation_basis": "Prior 3 month average * 1.05 (inflation factor)",
      "gl_account": "6100",
      "accrual_account": "2250",
      "status": "READY_TO_POST"
    },
    "interest_expense": {
      "estimate": "125,000",
      "calculation_basis": "Term loan balance * annual rate / 12",
      "balance": "15.2M",
      "rate": "4.75%",
      "gl_account": "6300",
      "accrual_account": "2300"
    },
    "payroll_payable": {
      "estimate": "234,500",
      "calculation_basis": "Bi-weekly payroll 5/31 not paid until 6/6",
      "gl_account": "5100",
      "liability_account": "2150"
    }
  },
  "reversals": {
    "entries_to_reverse": 8,
    "detail": [
      {
        "original_entry_date": "2026-04-30",
        "description": "April utilities accrual",
        "amount": "44,200",
        "reversal_effective_date": "2026-05-01",
        "status": "SCHEDULED"
      },
      {
        "original_entry_date": "2026-04-30",
        "description": "April interest accrual",
        "amount": "125,000",
        "reversal_effective_date": "2026-05-01",
        "status": "SCHEDULED"
      }
    ]
  },
  "journal_entries": [
    {
      "entry_number": "JE-2026-05-01",
      "description": "Reverse April accruals",
      "posting_date": "2026-05-01",
      "entries": [
        {
          "debit_account": "2250",
          "debit_amount": "44,200",
          "credit_account": "6100",
          "credit_amount": "44,200",
          "description": "Reverse utilities accrual"
        },
        {
          "debit_account": "2300",
          "debit_amount": "125,000",
          "credit_account": "6300",
          "credit_amount": "125,000",
          "description": "Reverse interest accrual"
        }
      ]
    },
    {
      "entry_number": "JE-2026-05-31",
      "description": "Record May accruals",
      "posting_date": "2026-05-31",
      "entries": [
        {
          "debit_account": "6100",
          "debit_amount": "45,300",
          "credit_account": "2250",
          "credit_amount": "45,300",
          "description": "Utilities accrual - May"
        },
        {
          "debit_account": "6300",
          "debit_amount": "125,000",
          "credit_account": "2300",
          "credit_amount": "125,000",
          "description": "Interest accrual - May"
        },
        {
          "debit_account": "5100",
          "debit_amount": "234,500",
          "credit_account": "2150",
          "credit_amount": "234,500",
          "description": "Payroll payable - 5/31"
        }
      ]
    }
  ],
  "close_checklist": {
    "items": [
      {
        "task": "Prior month accruals reversed",
        "status": "COMPLETED",
        "entries_posted": 8
      },
      {
        "task": "Month accruals recorded",
        "status": "COMPLETED",
        "entries_posted": 8
      },
      {
        "task": "GL reconciliations complete",
        "status": "PENDING",
        "note": "Run GL Reconciler agent"
      },
      {
        "task": "Trial balance reviewed",
        "status": "PENDING",
        "note": "Manual step"
      },
      {
        "task": "Financial statements generated",
        "status": "PENDING",
        "note": "Run reporting agent"
      }
    ]
  },
  "posting_summary": {
    "entries_posted": 10,
    "total_debits": "1,254,300",
    "total_credits": "1,254,300",
    "balance_check": "BALANCED"
  },
  "output_files": {
    "close_memo": "Month_End_Close_Summary_2026_05.docx",
    "entry_detail": "month_end_entries.xlsx",
    "close_checklist": "close_checklist.docx"
  }
}
```

## ข้อมูล MCP Servers ที่ต้องการ

| Server | ตัวแปร Environment | ข้อมูลที่ให้บริการ |
|--------|-------------------|------------------|
| **Internal GL** | `GL_MCP_URL` | GL detail, transaction history, balance queries, posting capability |

## Accrual Logic

Month-End Closer มี built-in accrual logic สำหรับ:

| Accrual Type | Calculation | GL Account | Accrual Account |
|-------------|-----------|-----------|-----------------|
| Utilities | 3-month average | 6100 | 2250 |
| Interest | Balance × rate / 12 | 6300 | 2300 |
| Payroll | Days-worked calculation | 5100 | 2150 |
| Rent | Fixed monthly | 6200 | 2240 |
| Supplies | Consumption estimate | 5200 | 2255 |

## Headless Month-End

Month-End Closer ทำงานเป็น headless system:

- ไม่มี interactive prompts
- ทำงานตามชุดกฎที่กำหนดไว้ (configured rules)
- Posts entries อัตโนมัติ
- Generates reports for review

## ความแตกต่างจาก Interactive Plugin

**Month-End Closer Plugin** (Interactive):
- ผู้ใช้เลือก accrual types
- Plugin ถามคำถามเกี่ยวกับ rates/assumptions
- ผู้ใช้เห็น entries ก่อนการโพสต์

**Month-End Closer Cookbook** (Managed):
- API ให้ period + accrual targets
- Post entries อัตโนมัติ
- ส่งออก close memo + entry list

## Integration with Close Process

Month-End Closer ใช้ได้กับ month-end workflow:

```
Day 1: Close subledgers → Day 2: Run Month-End Closer
       → Day 2: Run GL Reconciler (parallel)
       → Day 3: Review, approve entries
       → Day 3: Generate trial balance & statements
       → Day 4: Final approval & publish
```

---

**หมายเหตุ**: Month-End Closer ต้องการ GL write access — ใช้โดยบัญชีชั้นเก่ (senior accountant) สำหรับการกำกับดูแล
