---
title: Fund Administration (บริหารกองทุน)
parent: 04 Vertical Plugins
nav_order: 6
---

# Fund Administration — ศูนย์บันทึก

**Fund Administration** vertical ช่วย fund finance teams, administrators, ผู้ดูแล fund หนังสือบัญชี, NAV calculations, LP statement tie-outs, variance analysis

## ภาพรวม

เป้าหมาย: Automate fund finance — GL reconciliation, break trace, accrual management, NAV tie-outs, LP statement generation

**ผู้ใช้:**
- Fund administrators (internal operations)
- Fund finance teams
- Compliance teams
- Investment accountants
- LP accounting teams (finance verification)

**Workflows ที่สนับสนุน:**
- GL reconciliation (verify books match bank, custodian)
- Break trace (identify missing/unmatched trades, cash items)
- Accrual schedules (management fees, performance fees, expenses)
- Roll-forward (P&L, NAV movements quarter-to-quarter)
- Variance commentary (explain variances in financial reports)
- NAV tie-out (verify LP statements match NAV pack)

## Commands (Slash Commands)

*Note: Fund-admin ไม่มี `/commands` — ใช้ skills directly ที่เรียกจากการ upload ไฟล์ GL หรือ NAV pack*

**Skills invoked via upload:**

| Skill | ใช้ได้อย่างไร |
|-------|---------|
| `/gl-recon` | Upload GL export + bank statement → Match entries, flag breaks |
| `/break-trace` | Upload trade log + cash statement → Identify missing trades, timing differences |
| `/accrual-schedule` | Define fee structure, upload ledger → Generate accrual schedule |
| `/roll-forward` | Upload prior period financials + movement schedule → Update to current period |
| `/variance-commentary` | Upload budget vs actual + detailed GL → Generate variance explanations |
| `/nav-tieout` | Upload LP statement + NAV pack → Verify line-by-line, flag mismatches |

### ตัวอย่างการใช้ Commands

**GL Reconciliation:**
```
Upload: GL export + Bank statement (Q1 2025)
Output:
- Matched items: 847 (cash: $2.5B, securities: $850M)
- Unmatched GL entries: 3 (outstanding trades, timing)
- Unmatched bank items: 1 ($150K wire not yet posted to GL)
- Actions: Reverse trade entry for failed deal, post bank wire, investigate timing difference
```

**Break Trace:**
```
Upload: Trade log + settlement status report (Q1 2025)
Output:
- Settled trades: 182 (matched to GL)
- Unsettled/pending: 5 
  - Trade A: Failed settlement (counterparty default) → Need to reverse
  - Trade B: Settlement delayed by 1 day (broker system issue) → Post next day
- Cash breaks: $2.3K (likely rounding error across 5 trades)
- Recommendation: Reverse Trade A, monitor Trades B-D for settlement
```

**Accrual Schedule:**
```
Upload: Fee schedule + management agreement + NAV
Output:
- Management fee: 2.0% on AUM = $50M NAV × 2.0% / 4 quarters = $250K per quarter
- Performance fee: 20% of net gains (high-water mark) = $150K (based on Q1 realized gains)
- Admin expenses: Fixed $25K/quarter
- Total accrual: $425K Q1 2025
```

**NAV Tie-out:**
```
Upload: LP statement (draft) + NAV pack
Compare line-by-line:
LP: Johnson Family Fund
Status: ✓ Beginning capital matches prior quarter ✓
✓ Contributions ($500K called) match capital call register
✓ Allocated P&L ($50K gain) matches NAV pack (0.5% of $10M commitment)
✗ Distributions ($75K) — LP statement shows $75K, NAV pack shows $72K
   → Flag: Investigate $3K variance (rounding error vs actual distribution)
✓ Ending capital balance: matches within $100

Outcome: CONDITIONAL PASS — resolve $3K distribution variance before distribution
```

## Skills ที่มี

| ชื่อ Skill | ทำอะไร | วิธีหลัก |
|-----------|---------|---------|
| `gl-recon` | Match GL entries to bank/custodian statements, identify timing differences/breaks | Line-by-line comparison; reconciliation rules engine |
| `break-trace` | Identify trades not yet settled, cash items not yet cleared, timing breaks | Pulls trade log vs settlement status; flags by trade type |
| `accrual-schedule` | Generate management fee, performance fee, expense accruals based on master documents | Pulls fee schedule, NAV, realized/unrealized gains; calculates accruals |
| `roll-forward` | Update financials from prior period to current period (adjust for movements, new values) | Prior period template + movement schedule; updates cells |
| `variance-commentary` | Generate narrative explaining variances between budget/prior and actual | Variance analysis algorithm; templates for common explanations |
| `nav-tieout` | Verify LP statements match NAV pack; flag line-by-line discrepancies | Recomputes LP capital account from NAV inputs; compares |

## Hooks

**File:** `/fund-admin/hooks/hooks.json`

Status: **No hooks** — Skills are invoked on-demand via file upload

## MCP Connectors ที่เกี่ยวข้อง

| Connector | ใช้สำหรับ |
|-----------|---------|
| **Fund data API** (SS&C, Ultimus, etc.) | GL exports, NAV packs, LP statements |
| **Bank/Custodian APIs** | Bank statements, settlement details |
| **Deal database** | Trade confirmations, deal details |

## ความสัมพันธ์กับ Agent Plugins

**Fund Admin** standalone — does not consume from other verticals
(Provides data outputs that IB/PE may reference for portfolio analytics)

## ตัวอย่างการใช้งาน

### Workflow: Monthly GL Reconciliation (2 hours)

```
FUND: Apex Capital Partners II (PE fund, $500M committed capital)
PERIOD: January 2025

PROCESS:
1. FPA downloads GL extract from fund accounting system (SS&C)
   Exported: All GL accounts, JVs, consolidations
   
2. FPA downloads bank statement from custodian (BNY Mellon)
   Exported: Cash movements, wire confirmations, investment transactions
   
3. FPA uploads both to /gl-recon skill
   
4. Skill matches entries:
   GL CASH ACCOUNT: $2,543,200
   Bank statement: $2,545,600
   Difference: $2,400 (likely timing or fee)
   
5. Skill identifies unmatched items:
   GL Entry: "Investment purchase ABC Co" - $5M (dated 1/15)
   Bank: "Wire to Equity Trust for ABC" - $5M (dated 1/16 settlement)
   → Timing difference (expected, investment settlement delay)
   
   GL Entry: "Fee accrual - management" - $833K (standard monthly accrual)
   Bank: "Fee debit from AUM" - $835K (actual fee charged, includes rounding)
   → Minor variance ($2K) within tolerance
   
   GL Entry: Missing wire outbound $150K (dated 1/28, marked "pending")
   Bank: Wire posted 1/31 (2-day delay)
   → Expected; monitor for posting in Feb GL

6. Unmatched bank item:
   Bank shows: $1.2K fee debit (custodian quarterly fee)
   GL: Not yet posted
   → Expected; will post when invoice received (standard process)

7. Output report:
   RECONCILIATION SUMMARY
   GL Cash: $2,543,200
   Bank Cash: $2,545,600
   Difference: $2,400
   
   MATCHED ITEMS (Timing):
   ✓ Investment purchase/settlement (1-day timing)
   ✓ Management fee accrual (within tolerance)
   ✓ Wire pending (expected post in 2 days)
   
   UNMATCHED (Acceptable):
   ✓ Custodian fee not yet invoiced (will post when received)
   
   RECOMMENDATION: GL reconciliation APPROVED
   → No journal entries needed
   → Monitor wire posting in Feb GL (should clear timing difference)
   → Next month check custodian fee posting
```

### Workflow: Quarterly NAV Tie-Out (3 hours)

```
FUND: Apex Capital Partners II
PERIOD: Q1 2025 (quarter ending 3/31)

LPs: 45 investors, total committed capital $500M

PROCESS:

1. Fund admin generates LP statements (from fund accounting system)
   Each LP receives draft statement with:
   - Beginning capital (prior quarter ending balance)
   - Contributions (capital calls this period)
   - Distributions (exit proceeds, dividend distributions)
   - Allocated net income/(loss) [= LP% × (fund P&L - mgmt fee - fund expenses)]
   - Carried interest allocation (if any, crystallized this period)
   - Ending capital (new balance)

2. Fund admin uploads:
   - Draft LP statements (Excel file, all 45 LPs)
   - NAV pack (from accounting system)
   
3. Skill /nav-tieout loads both, tests each LP statement:

EXAMPLE LP: Cornerstone Pension Fund ($50M commitment)

Recompute LP capital from NAV pack:
   Beginning capital (per prior Q4 statement): $58.2M
   + Contributions (capital called Q1): $2.0M
   - Distributions (fund distributions Q1): $0M
   + Allocated P&L: 10% × ($8M realized gains + $12M unrealized gains - $1M mgmt fee - $0.2M fund expenses) 
      = 10% × $18.8M = $1.88M
   - Carried interest: $0M (not crystallized this period)
   = Ending capital should be: $62.08M

Compare to LP statement:
   Ending capital (on draft statement): $62.10M
   
Variance: +$20K (immaterial, likely rounding in intermediate calcs)
Status: PASS (within tolerance)

--- Repeat for all 45 LP statements ---

PROBLEMS FOUND (2 exceptions):

LP #12 - ABC Family Office ($8M commitment):
   Recomputed: $9.85M
   Statement shows: $9.72M
   Variance: -$130K
   Root cause: Statement calculated LP% as 1.6%, but actual commitment register shows 1.60% (rounding error downstream)
   Fix: Recalculate using correct 1.60% rate → $9.85M (now matches)
   Action: Update statement before distribution

LP #35 - XYZ Endowment ($12M commitment):
   Recomputed: $14.32M
   Statement shows: $14.28M
   Variance: -$40K
   Root cause: Prior period error (Q4 statement was off by -$40K, carried forward)
   Action: Correction needed in prior period close-out (coordinate with LP)

4. Output NAV tie-out report:
   TOTAL STATEMENTS TESTED: 45
   PASS (within tolerance): 43
   FLAG (variance >$25K): 2
   
   Summary:
   - ABC Family Office: Fix rounding error ($130K) → restate
   - XYZ Endowment: Resolve prior period carryforward ($40K) → coordinate with LP
   
   RECOMMENDATION: Hold Q1 distribution until above 2 items resolved
   (Both are data corrections, not actual fund P&L issues)
```

### Workflow: Variance Commentary (1 hour)

```
PERIOD: Q1 2025

Fund admin loads /variance-commentary
Inputs:
- Budget vs actual: Invested capital, P&L, fees, expenses
- Variance threshold: Flag items >$100K or >10%

Budget: Invest $50M this quarter
Actual: Invested $42.5M
Variance: -$7.5M (-15%)
Auto-commentary: "Investment pace below budget due to [1] delays in XYZ acquisition DD (2-week slip), [2] ABC IPO window closed, defer to Q2. Underlying deal sourcing on-track; expect catch-up in Q2."

Budget: Gross P&L $20M
Actual: Gross P&L $24.3M
Variance: +$4.3M (+21.5%)
Auto-commentary: "Outperformance driven by [1] earlier-than-expected exit of Fund I residual (realized gain $5M), [2] portfolio company XYZ better-than-plan FY results (unrealized gain uplift $2M), offset by [3] ABC Holdings write-down due to competitive pressure (-$2.7M)."

Budget: Management fees $2.5M
Actual: Management fees $2.48M
Variance: -$20K (-0.8%)
Auto-commentary: "Fee variance within tolerance (rounding on quarterly accrual). Year-to-date fees are on-budget."

Output: Variance commentary narrative (suitable for LP letter/reporting)
```

## Files

| Path | ทำอะไร |
|------|---------|
| `skills/gl-recon/SKILL.md` | GL reconciliation |
| `skills/break-trace/SKILL.md` | Trade/cash break identification |
| `skills/accrual-schedule/SKILL.md` | Fee/expense accrual generation |
| `skills/roll-forward/SKILL.md` | Period-to-period financials update |
| `skills/variance-commentary/SKILL.md` | Budget vs actual narrative |
| `skills/nav-tieout/SKILL.md` | LP statement verification |
| `.claude-plugin/plugin.json` | Plugin metadata |

---

**Fund Admin Vertical = Back-office**: GL, NAV, accruals, statements — foundation of accurate fund reporting
