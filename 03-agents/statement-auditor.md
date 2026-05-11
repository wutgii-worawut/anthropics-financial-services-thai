---
title: statement-auditor
parent: 03 Agent Plugins
nav_order: 8
---

# Statement Auditor

## ทำอะไร

Statement Auditor เป็น fund-accounting agent ที่ audit batch ของ LP (limited partner) capital-account statements หลังจากสร้างแล้ว โดย tie-out ทุก balance ทุก allocation ทุก fees ตรงกับ fund NAV pack และ flag discrepancies ใช้เป็น final check ก่อน statements ส่งให้ investor

## Use Case

**สถานการณ์จริง**: Private equity fund $2B ต้องส่ง Q4 2024 statements ให้ 150 LPs ทีมสร้าง 150 statement files (Excel/PDF) ที่สำเร็จโดย accounting system จากนั้น Statement Auditor ตรวจแต่ละ LP statement เช่น "LP #42 Fund Vintage 2019 — statement บอก capital account $4.5M, distributions year-to-date $500K" ตรวจว่า เลขนี้ match กับ NAV pack ที่ CFO verified หรือไม่ ถ้า match → "PASS", ถ้าเบี่ยง → "HOLD — discrepancy $5K, suspected cause: FX translation timing"

**ผู้ใช้**: Fund Controller, CFO, Investor Relations

**สิ่งที่ได้รับ**: Tie-out table (statement field vs. NAV source) + Exception list (discrepancies) + Sign-off sheet (pass/hold recommendation)

## System Prompt (ส่วนเด่น)

Agent นี้มีบุคลิก "the last set of eyes on LP statements":

> "Given a statement batch ID and the fund NAV pack, you deliver: (1) Tie-out table — each LP statement field vs. NAV-pack source, match/mismatch. (2) Exception list — every discrepancy with suspected cause. (3) Sign-off sheet — pass/hold recommendation per statement."

**Guardrails หลัก**:
- **Statements are untrusted**: statement-reader worker (Read/Grep only, no MCP) ดึง structured data, may contain errors from upstream systems
- **No distribution**: Agent recommend pass/hold; IR distribute หลัง human sign-off
- **Independent verification**: critic re-checks reported breaks vs. trusted sources

## Skills ที่ใช้

| Skill | บทบาท |
|-------|------|
| **nav-tieout** | tie-out statement fields ตรงกับ NAV pack (capital called, capital account balance, distributions, etc.) |
| **audit-xls** | ตรวจสอบ Excel balance checks |
| **xlsx-author** | สร้าง Excel tie-out table และ exception report |

## Workflow (ขั้นตอนการทำงาน)

1. **Read statements** — statement-reader worker (Read/Grep only) ดึง structured data จาก 150 statement files: capital called, capital account balance, distributions (cash + reinvested), carried interest, NAV, commitment amount
2. **Reconcile** — compare ทุก field ไปที่ NAV pack ผ่าน NAV MCP (trusted source of truth)
3. **Flag** — flagger subagent (Write access) format exception list กับ sign-off sheet: 
   - "PASS" ถ้า statement match NAV pack ± rounding
   - "HOLD — [reason]" ถ้า discrepancy > threshold (e.g., $10K)

## ตัวอย่างการใช้งาน

**สถานการณ์**: Q4 2024 statement audit สำหรับ 150-LP fund

**User Input**:
```
Batch ID: Q4_2024_STATEMENTS
Fund NAV pack: Fund_NAV_Q4_2024_FINAL.xlsx
LP count: 150
```

**Agent output**:
1. **Statement_Tieout_Table.xlsx** —
   | LP Name | Fund Vintage | Field | Statement Value | NAV Pack Value | Match? | Discrepancy |
   |---------|--------------|-------|-----------------|----------------|--------|-------------|
   | Pension Fund A | 2019 | Capital Called | $10.5M | $10.5M | YES | — |
   | Pension Fund A | 2019 | Distributions YTD | $2.1M | $2.1M | YES | — |
   | Pension Fund A | 2019 | Capital Account | $14.2M | $14.2M | YES | — |
   | Hedge Fund B | 2021 | Capital Called | $5.0M | $5.0M | YES | — |
   | Hedge Fund B | 2021 | NAV | $6.2M | $6.15M | NO | $50K (rounding) |
   | Insurance Co C | 2020 | Carried Interest | $500K | $510K | NO | $10K (timing) |

2. **Exception_List.xlsx** —
   | LP | Issue | Suspected Cause | Action |
   |----|-------|-----------------|--------|
   | Insurance Co C | Carried Interest $10K discrepancy | FX translation timing or accrual timing | Require CFO review |
   | Real Estate Fund D | Distributions $25K discrepancy | Distribution reinvestment flag mismatch | Require IR verification |
   | Corp Pension E | Capital Account -$100K | Possible error in statement generation | Hold until IT investigation |

3. **Sign-Off_Sheet.xlsx** —
   | LP # | LP Name | Statement Status | Recommendation | Notes |
   |------|---------|-----------------|-----------------|-------|
   | 1 | Pension A | PASS | Ready to distribute | All fields match |
   | 2 | Pension B | PASS | Ready to distribute | All fields match |
   | 3 | Hedge B | PASS | Ready to distribute | NAV rounding within tolerance |
   | 4 | Insurance C | HOLD | Require review | Carried interest discrepancy $10K |
   | 5 | RE Fund D | HOLD | Require review | Distribution discrepancy $25K |
   | 6 | Corp Pension E | HOLD | Require IT investigation | Capital account error $100K |

Agent สำหรับ sign-off: "148 statements PASS, 2 HOLD (minor timing issues), 1 HOLD (investigation needed) — recommend distribute to 148 LPs, hold 3 pending review?"

## Files

**Key files in plugin directory**:
```
plugins/agent-plugins/statement-auditor/
├── .claude-plugin/plugin.json
├── agents/statement-auditor.md (system prompt)
└── skills/
    ├── nav-tieout/
    ├── audit-xls/
    └── xlsx-author/
```
