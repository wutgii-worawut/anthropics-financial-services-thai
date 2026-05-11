---
title: Wealth Management (บริหารสินทรัพย์)
parent: 04 Vertical Plugins
nav_order: 5
---

# Wealth Management — ที่ปรึกษาทางการเงิน

**Wealth Management** vertical ช่วย wealth advisors, financial planners, portfolio managers ตรวจสอบผลงานลูกค้า ปรับสมดุลพอร์ตโฟลิโอ จัดทำแผนการเงิน และสร้างรายงานลูกค้า

## ภาพรวม

เป้าหมาย: Support advisor workflow — client review, portfolio rebalancing, financial planning, tax optimization, client reporting

**ผู้ใช้:**
- Private wealth advisors
- Financial planners
- Portfolio managers
- Relationship managers
- Client service teams

**Workflows ที่สนับสนุน:**
- Client review (portfolio performance vs benchmarks, risk assessment)
- Portfolio rebalancing (drift analysis, optimization)
- Tax-loss harvesting (identify realized losses, harvest opportunities)
- Financial planning (retirement projections, cash flow modeling)
- Investment proposal (recommendations for new allocations)
- Client reporting (quarterly/annual statements, performance attribution)

## Commands (Slash Commands)

| คำสั่ง | คำอธิบาย |
|---------|---------|
| `/client-review` | ตรวจสอบผลงาน: performance vs benchmark, risk assessment, action items |
| `/rebalance` | วิเคราะห์ drift — เพิ่ม/ลด allocations เพื่อกลับไปที่ target weights |
| `/tlh` | ระบุ tax-loss harvesting opportunities และ draft trades |
| `/financial-plan` | Create retirement plan, cash flow projection, goal-based planning |
| `/proposal` | Create investment proposal with recommended allocations, rationale |
| `/client-report` | Generate quarterly/annual report: performance, attribution, holdings |

### ตัวอย่างการใช้ Commands

**Client Review:**
```
/client-review [Client Name]
Output:
- Portfolio value: $2.5M (up 8% YTD vs benchmark +6%)
- Asset allocation: 60% equities, 35% bonds, 5% alternatives
- Risk profile: Moderate (volatility in-line with target)
- Underperformers: Tech (down 3% vs sector), emerging markets (flat)
- Actions: Rebalance bonds, discuss concentrated position in AAPL
```

**Rebalance:**
```
/rebalance [Client Name]
Target allocation: 60/35/5 (equities/bonds/alternatives)
Current: 62/33/5 (slight overweight to equities)

Recommended trades:
- Sell $30K US large-cap equities
- Buy $30K US intermediate-term bonds
- Result: Returns to target allocation
```

**Tax-Loss Harvesting:**
```
/tlh [Client Name] [Tax year]
Identified losses:
- ABC Fund: -$5K realized loss (hold-up 32 days)
- XYZ ETF: -$8K realized loss (no wash-sale conflict)
Total realizable losses: $13K
Tax benefit @ 35% marginal rate: $4,550
```

## Skills ที่มี

| ชื่อ Skill | ทำอะไร | วิธีหลัก |
|-----------|---------|---------|
| `client-review` | Quarterly/annual portfolio review — performance vs benchmark, risk metrics, dashboard | Gathers portfolio data, calculates returns, compares to benchmark indices |
| `portfolio-rebalance` | Analyze drift from target allocation, recommend trades to rebalance | Drift calculation (target vs actual), optimization (minimize tracking error/costs) |
| `tax-loss-harvesting` | Identify unrealized losses, check wash-sale rules, recommend harvest trades | Scans holdings, flags losses, calculates tax benefits |
| `financial-plan` | Retirement/goal-based planning — project cash flows, model growth scenarios | Simplified DCF-style modeling; retirement income analysis |
| `investment-proposal` | Create recommendation memo: current allocation vs proposed, rationale, risks | Template-driven; ties to client risk profile + goals |
| `client-report` | Quarterly/annual client-facing report: performance, holdings, attribution | Professional formatting; performance attribution; risk summary |

## Hooks

**File:** `/wealth-management/hooks/hooks.json`

Hooks ที่มี:
- **Quarterly**: Auto-generate client reviews for all managed accounts
- **Market correction >10%**: Flag rebalancing opportunities
- **Tax year-end**: Auto-identify TLH opportunities

## MCP Connectors ที่เกี่ยวข้อง

| Connector | ใช้สำหรับ |
|-----------|---------|
| **Portfolio data API** (Schwab, Fidelity, etc.) | Holdings, market values, cost basis |
| **Market data** | Current prices, benchmark indices (S&P 500, Bloomberg Aggregate Bond) |
| **Client CRM** | Client profile, risk tolerance, investment objectives |

## ความสัมพันธ์กับ Agent Plugins

**Wealth Management** light consumes from **Financial Analysis**:
- `/financial-plan` uses simplified 3-statement modeling
- `/rebalance` uses basic optimization logic

**Does not heavily depend on** IB/PE/Research plugins (different client base)

## ตัวอย่างการใช้งาน

### Workflow: Quarterly Client Review (30 min)

```
ADVISOR loads /client-review [Client: ABC Family Office, $2.5M portfolio]

PORTFOLIO SUMMARY:
- Market value: $2,542,300 (up 8.2% YTD vs S&P 500 +6.1%)
- Cost basis: $2,310,000
- Unrealized gain: $232,300 (10.1%)

PERFORMANCE ATTRIBUTION:
Equities (60% allocation, $1,525M):
  - US Large Cap (30%): +7.8% (vs S&P 500 +6.1%) — outperformed
  - US Mid/Small Cap (15%): +12.3% (vs Russell 2000 +11.2%) — outperformed
  - International (15%): -2.1% (vs MSCI EAFE -1.8%) — underperformed
  → Drag from emerging markets exposure (China regulatory headwinds)

Bonds (35% allocation, $889M):
  - US Intermediate (20%): +2.1% (vs Bloomberg Aggregate +2.0%) — in-line
  - High Yield (10%): +5.2% (vs HY Index +4.8%) — slight outperformance
  - International (5%): +1.8% (vs Barclays Global Aggregate +1.5%) — slight out

Alternatives (5% allocation, $127M):
  - Private Equity (3%): No current valuations
  - Real Estate (2%): +3.2% (based on NCREIF index proxy)

RISK ASSESSMENT:
- Volatility: 7.2% (annualized) vs target range 6.5-7.5% ✓ ON TARGET
- Concentration: AAPL = 8% of equities, MSFT = 6% → manageable
- Currency exposure: 15% international = 2.3% of total portfolio ✓

ACTIONS RECOMMENDED:
1. REBALANCE: Slight overweight to equities (62% vs 60% target)
   → Recommend selling $30K equities, buy $30K bonds
   
2. REDUCE EM exposure: China slump dragging returns
   → Current emerging markets = $150K (6% of equities)
   → Suggest trim to $100K (4% of equities) — redeploy to developed Asia
   
3. TAX LOSS HARVESTING: ABC Fund has -$5K loss
   → Recommend harvest + swap to XYZ Fund (similar exposure, 30-day hold-up)
   → Tax benefit: $1,750 @ 35% bracket
   
4. LIQUIDITY CHECK: Next planned spend = $75K (in 6 months)
   → Bond ladder mature: $60K due in 6 months ✓
   → No action needed

NEXT STEPS:
- Schedule with client to discuss EM reduction, rebalancing trades
- Execute rebalancing after client approval
- Harvest tax loss immediately (no wait-period conflict)
```

### Workflow: Tax-Loss Harvesting (20 min)

```
YEAR-END 2025 TAX PLANNING

Advisor loads /tlh [Client name]

CURRENT HOLDINGS WITH LOSSES:
Position | Holdings | Cost | Current | Loss | TLH Eligible?
ABC ETF | $150K | $155K | $148K | -$7K | YES (30+ days since purchase)
XYZ Fund | $200K | $212K | $195K | -$17K | NO (purchased 15 days ago, hold-up)
DEF Stock | $100K | $108K | $105K | -$3K | YES (down >2%)
GHI Bond | $75K | $75K | $72K | -$3K | YES
Total realizable losses: -$13K (ABC + DEF + GHI)

TAX IMPACT:
Current capital gains realized YTD: +$25K (from rebalancing trades)
With TLH harvest: +$25K - $13K = +$12K net realized gains
Tax at 20% long-term rate: $2,400
Tax savings from TLH: $2,600 (at 20% rate)

RECOMMENDED HARVEST + SWAP TRADES:
1. Sell ABC ETF $148K → Harvest -$7K loss
   Buy ABC Competitor Fund $148K (similar exposure, sector/style match)
   Hold-up period: 30 days (IRS wash-sale rule)

2. Sell DEF Stock $105K → Harvest -$3K loss
   Buy DEF Alternative (peer company, similar fundamentals)
   Hold-up period: 30 days

3. Sell GHI Bond $72K → Harvest -$3K loss
   Buy GHI Peer Bond (similar duration/credit, different issuer)
   Hold-up period: 30 days

TOTAL HARVESTED LOSSES: -$13K
TOTAL TAX SAVINGS: $2,600 (at 20% long-term cap gains rate)
```

### Workflow: Financial Plan (1 hour)

```
CLIENT: High-net-worth individual, age 52
Goals:
- Retire at 65 (13 years)
- Maintain current lifestyle: $200K/year spending (inflation-adjusted)
- Leave $1M to charity at death
- Travel budget: +$50K/year in retirement

Advisor loads /financial-plan [Client]

INPUTS:
Current portfolio: $2.5M
Current income: $400K/year (employment) + $50K (investment income)
Life expectancy: 95 (per mortality table)
Inflation: 2.5% annually
Expected returns: 6.5% stocks, 3.0% bonds (60/40 portfolio)

MODEL OUTPUTS:

WORKING YEARS (Age 52-65):
- Annual savings: $150K/year ($400K income - $200K spend - taxes/other)
- Portfolio growth: $2.5M → $5.8M (with $1.95M annual savings)
- Projected portfolio at retirement (age 65): $5.8M

RETIREMENT YEARS (Age 65-95):
- Annual spending: $250K nominal ($200K lifestyle + $50K travel)
- Inflation adjustment: Grows at 2.5%/year
- Investment returns: 6.5% (60/40 portfolio)

RETIREMENT CASH FLOW PROJECTION:
Age 65: Portfolio $5.8M, spend $250K, return $377K → Net +$127K (portfolio grows)
Age 75: Portfolio $8.2M, spend $320K (inflation), return $533K → Net +$213K
Age 85: Portfolio $10.5M, spend $410K, return $682K → Net +$272K
Age 95: Portfolio $12.1M, spend $526K → Bequest $1M+ to charity, residual $11M estate

OUTCOME: Plan is sustainable ✓ Portfolio never depleted through age 95

SENSITIVITY ANALYSIS:
- Market return down 1%: Portfolio at 95 = $10.2M (still sustainable)
- Market return down 2%: Portfolio at 95 = $8.5M (still sustainable, reduced legacy)
- Market return up 1%: Portfolio at 95 = $14.3M (surplus for legacy/extra spending)

RECOMMENDATIONS:
1. No change to current savings rate ($150K/year is sufficient)
2. Rebalance to 60/40 to lock in expected 6.5% returns
3. Review plan annually (market returns, spending changes, life expectancy updates)
4. At retirement, transition to 50/50 (reduce equity risk)
```

## Files

| Path | ทำอะไร |
|------|---------|
| `commands/client-review.md` | /client-review workflow |
| `commands/rebalance.md` | /rebalance workflow |
| `commands/tlh.md` | /tlh workflow — tax optimization |
| `commands/financial-plan.md` | /financial-plan workflow |
| `commands/proposal.md` | /proposal workflow |
| `commands/client-report.md` | /client-report workflow |
| `skills/client-review/SKILL.md` | Quarterly review |
| `skills/portfolio-rebalance/SKILL.md` | Drift analysis + rebalancing |
| `skills/tax-loss-harvesting/SKILL.md` | TLH identification |
| `skills/financial-plan/SKILL.md` | Retirement planning |
| `skills/investment-proposal/SKILL.md` | Recommendation memo |
| `skills/client-report/SKILL.md` | Quarterly/annual reporting |
| `.claude-plugin/plugin.json` | Plugin metadata |

---

**Wealth Management Vertical = Advisor toolkit**: Client review, portfolio optimization, financial planning, reporting
