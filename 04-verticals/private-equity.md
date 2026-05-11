---
title: Private Equity
parent: 04 Vertical Plugins
nav_order: 4
---

# Private Equity — ค้นหา แม่นให้ คัด ตัดสินใจ

**Private Equity** vertical ช่วย PE firms ค้นหา deal, screen deals, ประเมิน opportunities, จัดการ portfolio, วิเคราะห์ value creation

## ภาพรวม

เป้าหมาย: Accelerate PE deal lifecycle — sourcing → screening → diligence → investment decision → portfolio management → exit

**ผู้ใช้:**
- PE investors (partners, associates, analysts)
- Deal sourcing teams
- Investment committee members
- Portfolio company management

**Workflows ที่สนับสนุน:**
- Deal sourcing (company discovery, founder research, outreach)
- Deal screening (first-pass financial analysis, strategic fit)
- Due diligence preparation (data gathering, red flags)
- Investment committee (IC) memo creation
- Unit economics analysis (customer cohort analysis, pricing power)
- Value creation planning (3-year plan, add-on strategy)
- Portfolio monitoring (returns analysis, exit scenarios)
- AI readiness assessment (PE as digital acceleration platform)

## Commands (Slash Commands)

| คำสั่ง | คำอธิบาย |
|---------|---------|
| `/source` | ค้นหา target companies (sector, size, geography) → CRM check → draft founder outreach |
| `/screen-deal` | First-pass analysis: financials, strategic fit, valuation, red flags |
| `/dd-prep` | Due diligence prep — data checklist, document request list, timeline |
| `/dd-checklist` | DD checklist: legal, tax, finance, ops, customer/supplier concentration |
| `/ic-memo` | Investment committee memo — investment thesis, financials, synergies, risks, recommendation |
| `/unit-economics` | Model customer unit economics: CAC, LTV, cohort analysis, payback period |
| `/value-creation` | 3-year value creation plan: revenue growth, margin expansion, add-ons, exit scenario |
| `/returns` | Exit analysis — IRR, MoIC (Multiple on Invested Capital) under different exit scenarios |
| `/portfolio` | Monitor portfolio: current holdings, performance vs plan, risk flags |
| `/ai-readiness` | Assess AI opportunity: current AI maturity, implementation roadmap, value creation potential |

### ตัวอย่างการใช้ Commands

**Source:**
```
/source "B2B SaaS" "Texas" "$10-50M revenue" "founder-owned"
Output: 
- List 8-12 companies matching criteria
- CRM check: which we've contacted before
- Draft cold emails to founders (ready to send)
```

**Screen Deal:**
```
/screen-deal "ABC SaaS Inc" $25M revenue $5M EBITDA 20% growth
Output:
- Valuation: $100-125M (5.0x EBITDA @ 4.0-5.0x multiple)
- Returns: $100M investment → $180M exit (4.3 years) = 18% IRR
- Fit: Platform play in vertical SaaS (✓ fits thesis)
- Risks: Customer concentration, product roadmap maturity
```

**IC Memo:**
```
/ic-memo TARGET: ABC Manufacturing $200M EBITDA
Thesis: Consolidate fragmented industrial market (6k mom-and-pop competitors)
Investment: $500M equity @ 4.0x EBITDA (acquisition price $800M + fees)
Returns: 4-year hold, 15-20% IRR (exit at 6.5x EBITDA = $1.3B)
Synergies: Platform playbook, add-on acquisitions, operational improvements
```

## Skills ที่มี

| ชื่อ Skill | ทำอะไร | วิธีหลัก |
|-----------|---------|---------|
| `deal-sourcing` | Company discovery (web search, industry reports), CRM check, founder outreach | Web research + Gmail/Slack check + personalized email drafting |
| `deal-screening` | Quick financial analysis (valuation multiples, growth rates, fit to thesis) | Financial-analysis skill consumption (comps, simple DCF) |
| `dd-meeting-prep` | Schedule DD meetings, build information request list, track documents | Calendar integration + data organization |
| `dd-checklist` | Comprehensive DD guide: legal, tax, financial, operational, customer/supplier, management | Structured checklist template |
| `ic-memo` | Investment committee memo: thesis, financials, synergies, risks, returns, recommendation | Template-driven; integrates valuation from financial-analysis |
| `unit-economics` | Model customer economics: CAC, LTV, retention, cohort analysis, payback | Simplified cohort modeling |
| `value-creation-plan` | 3-year plan: revenue growth initiatives, margin expansion, add-on strategy, exit plan | Scenario modeling (conservative, base, upside) |
| `returns-analysis` | Model acquisition returns: IRR, MoIC under different exit multiples/timing | Sensitivity analysis (exit multiple, hold period) |
| `portfolio-monitoring` | Track portfolio company performance vs plan, flag performance issues | Aggregates management team reports, highlights variances |
| `ai-readiness` | Assess AI opportunity in target or portfolio company | Structured evaluation: current AI use, gaps, implementation potential, value impact |

## Hooks

**File:** `/private-equity/hooks/hooks.json`

Hooks ที่มี:
- **New investment** → Auto-populate portfolio tracker
- **Portfolio review** → Flag companies at risk of missing plan
- **Quarterly**: Auto-generate returns analysis

## MCP Connectors ที่เกี่ยวข้อง

| Connector | ใช้สำหรับ |
|-----------|---------|
| **Gmail** | Founder outreach history, deal correspondence |
| **Slack** | Internal IC discussions, deal commentary |
| **LinkedIn** | Management team profiles, founder backgrounds |
| **D&B / Crunchbase** | Company financials, deal history, investor networks |
| **CRM** (if configured) | Deal pipeline, relationship tracking |

## ความสัมพันธ์กับ Agent Plugins

**Private Equity** consumes from **Financial Analysis**:
- `/screen-deal` ใช้ `comps-analysis`, `dcf-model`
- `/ic-memo` ใช้ `dcf-model`, `3-statement-model`, `lbo-model`
- `/returns` ใช้ sensitivity analysis from valuation

**Provides to Portfolio Management agent:**
- Investment memo templates
- Value creation tracking

## ตัวอย่างการใช้งาน

### Workflow: Deal Lifecycle (6 weeks)

```
WEEK 1-2: SOURCING

1. Partner loads /source
   Criteria: Industrial services, $15-50M revenue, 
            Midwest/Southeast, founder-owned
   
2. Plugin identifies 12 companies:
   - ABC Services (Indianapolis, $28M rev, 20% EBITDA margin)
   - XYZ Industries (Nashville, $35M rev, 18% EBITDA margin)
   - DEF Maintenance (Memphis, $22M rev, 15% EBITDA margin)
   ... (others)

3. Plugin checks CRM:
   - ABC: "New" (no prior contact)
   - XYZ: "Existing" (met founder 6 months ago, passed then)
   - DEF: "Existing" (recent conversation with CEO)

4. Plugin drafts emails for "New" targets:
   "Hi [Founder], I came across [Company] and was impressed 
    by your market position in [Sector]. We work with platform 
    companies in your space. Would you be open to a brief conversation 
    about your business and what you're building? [Partner name]"

WEEK 3: SCREENING

5. Partner reviews list, selects ABC for deeper dive
6. Partner loads /screen-deal ABC Services
   
7. Plugin gathers public info:
   - Est. revenue: $28M (growing 18% annually)
   - Est. EBITDA: ~$4.2M (15% margin, typical for sector)
   - Employees: 180
   - Market: Highly fragmented (6000+ competitors)
   
8. Plugin analyzes:
   Valuation: $28M revenue × 4.0x EBITDA = $16.8M purchase price
             (assumes $4.2M EBITDA; comparable deals trade 3.5-5.0x)
   
   Returns: $15M equity investment → hold 5 years
           Synergy case: Add bolt-ons, improve margins 15%→18%
           Exit: 5.5x EBITDA = $42M exit value
           IRR: 24% (MoIC 2.8x)
   
   Fit: ✓ Platform thesis (consolidation play)
        ✓ Recurring revenue (contracts)
        ✗ Management team (founder-only, scalability risk)
   
   Red flags: 
   - Single customer >20% revenue?
   - Key employee retention post-close?
   - Integration of acquired add-ons?

WEEK 4: DUE DILIGENCE INITIATION

9. Partner loads /dd-prep ABC Services
   
   Plugin creates:
   - Document request list (3Y financials, customer contracts, 
     employee agreements, real estate leases, insurance, litigation)
   - DD meeting schedule (finance, ops, management, customer calls)
   - Timeline: 6-week DD starting Week 4
   
10. Partner loads /dd-checklist
    
    Comprehensive checklist:
    FINANCIAL: Revenue recognition, EBITDA add-backs, working capital
    LEGAL: Contracts (customers, suppliers), IP ownership, litigation
    TAX: Tax history, exposure, synergy tax savings potential
    OPS: Capex needs, CapEx % of revenue, maintenance vs growth spend
    CUSTOMERS: Top 10 customers = ?% of revenue, contract terms, churn
    SUPPLIERS: Top 5 suppliers, switching costs, sole sourcing risks
    MGMT: Key person risk, org structure, succession plan
    MARKET: Competitive positioning, pricing power, TAM expansion

WEEK 5-6: INVESTMENT COMMITTEE MEMO

11. Deal team assembles IC memo
    Partner loads /ic-memo ABC Services
    
    MEMO OUTLINE:
    
    INVESTMENT THESIS:
    - Industrial services fragmentation = consolidation opportunity
    - ABC = quality platform (founder-led, growing 18%, 15% margins)
    - Playbook: Buy platform → Add 3-5 tuck-in acquisitions → Exit at 5.5x
    
    FINANCIAL SUMMARY:
    LTM Revenue: $28M (growing 18%)
    LTM EBITDA: $4.2M (15% margin)
    Purchase price: $16.8M (4.0x EBITDA)
    Our equity: $15M (89% leverage)
    
    3-YEAR PLAN:
    Year 1: Platform integration, add 1 tuck-in ($3M revenue)
           Revenue $30.5M, EBITDA $4.8M (15.7%)
    Year 2: Add 2nd bolt-on ($5M revenue acquisition)
           Revenue $35M, EBITDA $5.6M (16%)
    Year 3: Add 3rd bolt-on, margin improvement to 17%
           Revenue $42M, EBITDA $7.1M (17%)
    
    EXIT SCENARIO:
    Assume 5.5x EBITDA multiple (in-line with comparable sales)
    Year 3 EBITDA $7.1M × 5.5x = $39.1M enterprise value
    Less net debt $1.8M = equity proceeds $37.3M
    On $15M equity investment = 2.5x MoIC, 20% IRR
    
    RISKS:
    - Management team (founder-led, scalability post-add-ons)
    - Customer concentration (TBD in DD)
    - Acquisition integration (tuck-in success rate)
    - Market cyclicality (industrial downturn risk)
    
    RECOMMENDATION:
    PROCEED with DD (conditional on management depth + top 10 customer check)

12. IC votes → Approval to proceed with DD

WEEK 7-12: DUE DILIGENCE → INVESTMENT DECISION

13. 6-week DD process → Final offer negotiation
    
14. Deal closes → Plug into /portfolio tracker
```

### Workflow: Returns Analysis (2 hours)

```
Partner loads /returns
Target: ABC Services (now owned 2 years)
Current performance:
- Revenue: $36M (vs plan $35M) ✓
- EBITDA: $5.8M vs plan $5.6M ✓
- Debt paydown: $3M (accelerated vs plan)

Plugin models exit scenarios:
Exit in Year 3:
- Exit multiple 5.0x (downside): $29M equity proceeds (1.9x MoIC, 18% IRR)
- Exit multiple 5.5x (base): $39M equity proceeds (2.6x MoIC, 21% IRR)
- Exit multiple 6.0x (upside): $43M equity proceeds (2.9x MoIC, 23% IRR)

Output: On-track for 20%+ IRR assuming base case exit in Year 3
```

## Files

| Path | ทำอะไร |
|------|---------|
| `commands/source.md` | /source workflow — deal discovery |
| `commands/screen-deal.md` | /screen-deal workflow — quick analysis |
| `commands/dd-prep.md` | /dd-prep workflow — due diligence setup |
| `commands/dd-checklist.md` | /dd-checklist workflow |
| `commands/ic-memo.md` | /ic-memo workflow — investment decision |
| `commands/unit-economics.md` | /unit-economics workflow |
| `commands/value-creation.md` | /value-creation workflow |
| `commands/returns.md` | /returns workflow — IRR analysis |
| `commands/portfolio.md` | /portfolio workflow — monitoring |
| `commands/ai-readiness.md` | /ai-readiness workflow |
| `skills/deal-sourcing/SKILL.md` | Company discovery |
| `skills/deal-screening/SKILL.md` | Quick screening |
| `skills/dd-meeting-prep/SKILL.md` | DD logistics |
| `skills/dd-checklist/SKILL.md` | Comprehensive checklist |
| `skills/ic-memo/SKILL.md` | Investment memo |
| `skills/unit-economics/SKILL.md` | Customer cohort analysis |
| `skills/value-creation-plan/SKILL.md` | 3-year plan |
| `skills/returns-analysis/SKILL.md` | IRR/MoIC modeling |
| `skills/portfolio-monitoring/SKILL.md` | Performance tracking |
| `skills/ai-readiness/SKILL.md` | AI assessment |
| `.claude-plugin/plugin.json` | Plugin metadata |

---

**PE Vertical = Deal pipeline**: Source → Screen → DD → IC memo → Portfolio track → Exit
