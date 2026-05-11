---
title: GL Reconciler
parent: 06 Managed Agent Cookbooks
nav_order: 9
---

# GL Reconciler Cookbook

## โครงสร้าง Agent

**GL Reconciler** เป็น managed agent ที่ประสานรับหนึ่งหรือชุดของหนังสือแยกประเภท (subledgers) เข้ากับบัญชีแยกประเภท (GL) และระบุความแตกต่าง

```yaml
model: claude-opus-4-7
tools:
  - agent_toolset_20260401 (read, grep, glob เท่านั้น - no write)
  - mcp_toolset: internal-gl (Internal GL Server)
  - mcp_toolset: subledger (Subledger Data)
```

## Subagents

GL Reconciler ประกอบด้วย 3 subagents ที่ไม่มี Write permission:

| Subagent | บทบาท | หน้าที่ |
|----------|------|--------|
| `reader` | Data collection | อ่าน GL balances, subledger data (AP, AR, Fixed Assets, etc.) |
| `critic` | Analysis & exceptions | Identify reconciling items, unexplained differences, aging issues |
| `resolver` | Reconciliation | Propose journal entries, document reconciliation logic |

## Important: Orchestrator Pattern

GL Reconciler **orchestrator agent ไม่มี Write permission**:
- Read-only access ไป internal GL และ subledgers
- Delegates to read-only criticisms & analysis
- Hands off journal entry preparation to external resolver (manual process)

### Subagent Hierarchy

```
GL Reconciler (orchestrator - read only)
  ├─ reader (read-only)
  ├─ critic (read-only)
  └─ resolver (coordination only - no Write)
```

## GL Reconciliation Workflow

GL Reconciler ทำหน้าที่เป็น automated reconciliation controller:

1. **Data Extraction** — reader pulls GL detail + subledger trial balances
2. **Analysis** — critic identifies exceptions, aging issues, unusual items
3. **Exception Resolution** — resolver proposes entries, documents rationale
4. **Handoff** — prepares package for human accountant to post

## การ Deploy

### 1. เตรียมสภาพแวดล้อม

```bash
export GL_MCP_URL="https://your-internal-gl-server"
export SUBLEDGER_MCP_URL="https://your-subledger-server"
```

### 2. Deploy Agent

```bash
claude-code deploy ./managed-agent-cookbooks/gl-reconciler/agent.yaml
```

### 3. เรียกใช้ via API

```bash
curl -X POST https://api.claude.ai/v1/agents/gl-reconciler/run \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "period": "2026-05",
    "subledgers": ["accounts_payable", "accounts_receivable", "fixed_assets"],
    "gl_accounts": ["1100", "1200", "1500", "2100"],
    "tolerance_amount": 1000
  }'
```

## โครงสร้าง Output

```json
{
  "status": "completed",
  "period": "2026-05",
  "reconciliation_summary": {
    "subledgers_reconciled": 3,
    "total_exceptions_found": 5,
    "total_reconciled": true,
    "total_difference": 0
  },
  "subledger_reconciliations": [
    {
      "subledger": "Accounts Payable",
      "gl_account": "2100",
      "gl_balance": "12,543,210",
      "subledger_balance": "12,541,850",
      "difference": "1,360",
      "status": "OUT_OF_BALANCE",
      "exceptions": [
        {
          "type": "Unposted Invoice",
          "vendor": "Supplier XYZ",
          "amount": "1,200",
          "date": "2026-05-31",
          "action": "Post accrual JE"
        },
        {
          "type": "Duplicate Invoice",
          "vendor": "Vendor ABC",
          "amount": "160",
          "action": "Reverse duplicate GL entry"
        }
      ]
    },
    {
      "subledger": "Accounts Receivable",
      "gl_account": "1100",
      "gl_balance": "8,234,560",
      "subledger_balance": "8,234,560",
      "difference": 0,
      "status": "RECONCILED"
    },
    {
      "subledger": "Fixed Assets",
      "gl_account": "1500",
      "gl_balance": "45,678,900",
      "subledger_balance": "45,678,900",
      "difference": 0,
      "status": "RECONCILED"
    }
  ],
  "aging_analysis": {
    "ap_aging": [
      {
        "bucket": "Current",
        "balance": "9.2M",
        "item_count": 234
      },
      {
        "bucket": "30+ days",
        "balance": "2.1M",
        "item_count": 18,
        "concern": "Unusual - requires review"
      }
    ],
    "ar_aging": [
      {
        "bucket": "Current",
        "balance": "6.8M",
        "item_count": 567
      },
      {
        "bucket": "90+ days",
        "balance": "0.3M",
        "item_count": 2,
        "concern": "Monitor for collection"
      }
    ]
  },
  "proposed_journal_entries": [
    {
      "description": "Accrue unposted invoice from Supplier XYZ",
      "debit_account": "5000",
      "debit_amount": "1,200",
      "credit_account": "2100",
      "credit_amount": "1,200",
      "reference": "Supplier XYZ Invoice #INV-2026-0512"
    },
    {
      "description": "Reverse duplicate fixed asset capitalization",
      "debit_account": "3000",
      "debit_amount": "160",
      "credit_account": "1500",
      "credit_amount": "160",
      "reference": "Duplicate equipment entry"
    }
  ],
  "reconciliation_notes": [
    "AP reconciliation completed with 2 exceptions requiring journal entries",
    "AR aging shows one small item >90 days — normal"
  ],
  "output_files": {
    "reconciliation_report": "GL_Reconciliation_2026_05.docx",
    "exception_detail": "exceptions_detail.xlsx",
    "proposed_entries": "proposed_je.xlsx"
  }
}
```

## ข้อมูล MCP Servers ที่ต้องการ

| Server | ตัวแปร Environment | ข้อมูลที่ให้บริการ |
|--------|-------------------|------------------|
| **Internal GL** | `GL_MCP_URL` | GL account balances, detail, posting history |
| **Subledger** | `SUBLEDGER_MCP_URL` | AP, AR, fixed assets, payroll, subledger trial balances |

## Control Environment

GL Reconciler operates within strong control framework:

- **Read-Only Processing** — No write access ให้ GL หรือ subledgers
- **Proposal Only** — Proposed entries ต้องผ่าน human review & approval
- **Audit Trail** — ทั้งหมด exceptions documented เพื่อ audit purposes
- **Segregation of Duties** — Different user posts proposed entries

## ความแตกต่างจาก Interactive Plugin

**GL Reconciler Plugin** (Interactive):
- ผู้ใช้เลือก subledgers และ GL accounts
- Plugin ถามคำถามเกี่ยวกับ tolerance levels
- Reconciliation ทำขั้นต่อ ขั้นต่อ

**GL Reconciler Cookbook** (Managed):
- API ให้ period + list of subledgers
- Run reconciliation อัตโนมัติ
- ส่งออก report + proposed entries for approval

## Month-End Integration

GL Reconciler ใช้ได้ดีใน month-end close process:

1. Day 1-2: Close subledgers, GL close
2. Day 2-3: Run GL Reconciler ขนานกับ manual processes
3. Day 3: Review exceptions, approve proposed entries
4. Day 4: Post approved entries, final close

---

**หมายเหตุ**: GL Reconciler ต้องหา human accountant สำหรับการ review และ posting — ไม่มี automaton complete
