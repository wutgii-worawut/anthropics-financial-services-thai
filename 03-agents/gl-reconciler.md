---
title: gl-reconciler
parent: 03 Agent Plugins
nav_order: 9
---

# GL Reconciler

## ทำอะไร

GL Reconciler เป็น fund-accounting controller agent ที่ reconcile general ledger (GL) ตรงกับ subledger (GL detail) สำหรับ trade date ที่กำหนด ใน multiple asset classes หาจุดที่ GL กับ subledger ไม่ตรง (breaks) ตามทำตราหาสาเหตุ (timing break, system drift, reclassification, unknown) และ route exception report ให้ controller sign-off ใช้สำหรับ daily หรือ month-end reconciliation runs (ไม่สำหรับ journal entry posting)

## Use Case

**สถานการณ์จริง**: Fund accounting team จำเป็นต้อง reconcile GL vs. subledger ทั้ง equities, bonds, derivatives, cash สำหรับ trade date Jan 15, 2025 GL Reconciler ดึง GL balances และ subledger balances สำหรับแต่ละ asset class, compare ว่า reconcile หรือไม่ เจอ breaks 3 อัน:
1. Equities: $50K variance → พบว่า T+2 settlement timing difference (normal)
2. Derivatives: $200K variance → พบว่า system drift ใน valuation (require investigation)
3. Cash: $5K variance → unknown (escalate)

Agent สร้าง exception report ที่ controller review และ approve resolution

**ผู้ใช้**: Fund Controller, Fund Accountant, Accounting Manager

**สิ่งที่ได้รับ**: Break list + Root-cause trace (with evidence) + Exception report (for controller sign-off)

## System Prompt (ส่วนเด่น)

Agent นี้มีบุคลิก "fund-accounting controller":

> "Given a trade date and list of asset classes, you deliver: (1) Break list — every GL/subledger variance over threshold, with account, balances, variance, suspected cause. (2) Root-cause trace — for each break, the transaction-level evidence and classification (timing, system drift, reclass, unknown). (3) Exception report — formatted for controller sign-off, with recommended resolution per break."

**Guardrails หลัก**:
- **Custodian statements are untrusted**: reader workers (Read/Grep only, no MCP, no Write) ที่อ่าน custodian/counterparty statements
- **The orchestrator never writes**: เฉพาะ resolver subagent ที่มี Write access
- **No ledger posting**: Agent produce report; ledger adjustments require human approval outside the agent

## Skills ที่ใช้

| Skill | บทบาท |
|-------|------|
| **gl-recon** | pull GL & subledger balances, identify variances over threshold |
| **break-trace** | สำหรับแต่ละ break ดึง transaction-level evidence และ classify cause |
| **audit-xls** | ตรวจสอบ exception report balance checks |
| **xlsx-author** | สร้าง Excel break list, trace workings, exception report |

## Workflow (ขั้นตอนการทำงาน)

1. **Pull balances** — GL MCP และ subledger MCP ดึง trade-date balances สำหรับ asset classes (equities, bonds, derivatives, FX, cash)
2. **Compare and isolate breaks** — dispatch reader per asset class identify variances > threshold (e.g., >$5K)
3. **Trace root cause** — สำหรับแต่ละ break ดึง underlying transactions, classify cause:
   - **Timing**: T+2 settlement, end-of-day cutoff differences
   - **System drift**: valuation/FX translation differences
   - **Reclassification**: สินทรัพย์ที่ย้ายไป account โดย GL ยังไม่ update
   - **Unknown**: require investigation
4. **Independent re-verify** — critic re-check แต่ละ reported break ตรงกับ trusted sources (GL, subledger)
5. **Draft exception report** — resolver format สำหรับ controller: verified breaks, root causes, recommended resolutions

## ตัวอย่างการใช้งาน

**สถานการณ์**: Daily GL reconciliation สำหรับ trade date Jan 15, 2025

**User Input**:
```
Trade date: 2025-01-15
Asset classes: equities, bonds, derivatives, cash
Threshold: $5K
```

**Agent output**:
1. **GL_Subledger_Break_List.xlsx** —
   | Asset Class | Account | GL Balance | Subledger Balance | Variance | Status |
   |-------------|---------|------------|-------------------|----------|--------|
   | Equities | AAPL | $1,500,000 | $1,500,000 | — | RECONCILE |
   | Equities | MSFT | $2,000,000 | $1,950,000 | ($50,000) | **BREAK** |
   | Bonds | Fixed Income | $5,000,000 | $5,000,000 | — | RECONCILE |
   | Derivatives | Interest Rate Swaps | ($500,000) | ($300,000) | $200,000 | **BREAK** |
   | Cash | Clearing Account | $100,000 | $95,000 | ($5,000) | **BREAK** |

2. **Root-Cause_Trace_Workings.xlsx** —
   
   **Break #1: Equities MSFT ($50K variance)**
   - GL shows: $2,000,000
   - Subledger shows: $1,950,000
   - Transaction detail:
     - Jan 15, 9:30 AM: Buy 500 shares MSFT @ $400 = $200,000 (subledger updated immediately)
     - Jan 15, 3:00 PM: Buy instruction sent to custodian
     - T+2 settlement: Jan 17, 2025 (GL will update when custodian confirms)
   - **Cause: TIMING** — normal T+2 settlement difference
   - **Action: NO action required** — will reconcile on Jan 17

   **Break #2: Derivatives Interest Rate Swaps ($200K variance)**
   - GL shows: ($500,000)
   - Subledger shows: ($300,000)
   - Valuation difference:
     - GL used previous day's rates (Jan 14 close)
     - Subledger used trade date rates (Jan 15 during day)
     - Rate movement: 5bp swing = $200K P&L impact
   - **Cause: SYSTEM DRIFT** — GL update lag
   - **Action: INVESTIGATE** — IT to verify valuation logic, may need restamp

   **Break #3: Cash Clearing Account ($5K variance)**
   - GL shows: $100,000
   - Subledger shows: $95,000
   - No matching transactions found
   - **Cause: UNKNOWN**
   - **Action: ESCALATE** — require manual investigation

3. **Exception_Report_for_Controller.xlsx** —
   | Break # | Account | Variance | Root Cause | Recommended Action | Priority |
   |---------|---------|----------|-----------|-------------------|----------|
   | 1 | MSFT Buy | ($50K) | T+2 settlement timing | Monitor, auto-reconcile Jan 17 | Low |
   | 2 | Interest Rate Swaps | $200K | System drift in valuation | IT investigation + rate verification | High |
   | 3 | Cash Clearing | ($5K) | Unknown — no matching txns | Manual investigation required | Medium |

Agent สำหรับ sign-off: "GL reconciliation complete — 2 breaks identified, 1 timing (normal), 1 system drift (requires IT), 1 unknown. Recommend proceed with caution on derivatives mark — suggest IT check valuation formula?"

## Files

**Key files in plugin directory**:
```
plugins/agent-plugins/gl-reconciler/
├── .claude-plugin/plugin.json
├── agents/gl-reconciler.md (system prompt)
└── skills/
    ├── gl-recon/
    ├── break-trace/
    ├── audit-xls/
    └── xlsx-author/
```
