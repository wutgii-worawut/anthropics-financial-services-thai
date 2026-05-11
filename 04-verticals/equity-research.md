---
title: Equity Research (วิจัยหุ้น)
parent: 04 Vertical Plugins
nav_order: 3
---

# Equity Research — ห้องวิจัยหุ้น

**Equity Research** vertical ช่วย equity analysts, research teams สร้าง research reports, ติดตามอัปเดต earnings, วิเคราะห์ catalysts, และจัดการ investment thesis

## ภาพรวม

เป้าหมาย: Support research workflow — เพิ่มความเร็ว coverage initiation, earnings forecasting, model updates, portfolio monitoring

**ผู้ใช้:**
- Equity analysts (buyside, sellside)
- Research managers
- Portfolio managers monitoring coverage
- Sector specialists

**Workflows ที่สนับสนุน:**
- Initiating coverage on new stocks (5-task process → full research report)
- Earnings preview (model expectations vs consensus)
- Earnings analysis (actual results vs model vs guidance)
- Model updates (update 3-statement financials, DCF with new guidance)
- Catalyst identification and calendar
- Thesis tracking (investment thesis, key risks, monitoring)
- Sector overview (peer comparisons, industry trends)
- Morning note creation (daily research summary)
- Idea generation (stock screener output analysis)

## Commands (Slash Commands)

| คำสั่ง | คำอธิบาย |
|---------|---------|
| `/initiate` | Start initiating coverage — 5 tasks: research, model, thesis, report, publication |
| `/earnings` | Analyze actual earnings results vs model forecast vs consensus guidance |
| `/earnings-preview` | Preview earnings — model expected results, sensitivity to upside/downside cases |
| `/model-update` | Update financial model with new guidance, revised assumptions, new forecast |
| `/catalysts` | Build catalyst calendar — upcoming events (earnings, product launch, FDA approval) |
| `/thesis` | Document investment thesis: bull case, bear case, risks, monitoring checklist |
| `/morning-note` | Create daily research summary — overnight news, market setup, key stories |
| `/sector` | Analyze sector — peer comparisons, industry trends, group recommendation |
| `/screen` | Analyze stock screener results — which stocks meet valuation/growth criteria |

### ตัวอย่างการใช้ Commands

**Initiate Coverage:**
```
/initiate NVIDIA
1. Research: Gathers business model, competitive positioning
2. Model: Creates 3-statement + DCF (10Y GPU TAM growth)
3. Thesis: Bull case on AI adoption, bear case on competition
4. Report: Writes 20-page initiating report
5. Publish: Readies for distribution (compliance review)
```

**Earnings Preview:**
```
/earnings-preview TSLA 2025Q1
Model expects:
- Revenue: $24.5B (vs consensus $23.8B)
- EPS: $0.82 (vs consensus $0.75)
- Margin: 18.5% (vs historical 15%)
Setup: Positive surprise likely; watch guidance tone
```

**Model Update:**
```
/model-update Apple
Input: New guidance $240-250B revenue (up from $235B model)
Updates: 
- FY25 revenue → midpoint $245B
- FCF grows with higher revenue
- Terminal value increases by 3%
- Price target: $215 (up from $205)
```

## Skills ที่มี

| ชื่อ Skill | ทำอะไร | วิธีหลัก |
|-----------|---------|---------|
| `initiating-coverage` | 5-step process: research company, build model, document thesis, write report, prepare for publication | Structured workflow; gathers business model, competitive analysis, financial model, DCF valuation |
| `earnings-analysis` | Compare actual results vs model vs consensus; identify surprises, update outlook | Pulls latest earnings, compares line-by-line, flags significant deviations |
| `earnings-preview` | Pre-earnings model: consensus vs "our" forecast, sensitivity to upside/downside case | References financial-analysis 3-statement skill; models EPS sensitivity |
| `thesis-tracker` | Document investment thesis: bull/bear cases, key risks, monitoring checklist | Structured template; tracks thesis evolution; flags thesis breaches |
| `model-update` | Update existing financial model with new guidance, revised assumptions | Links to financial-analysis 3-statement + DCF; recalculates valuations |
| `catalyst-calendar` | Build event calendar: earnings dates, product launches, regulatory milestones, conference attendance | Sources: company calendar, industry events, regulatory schedules |
| `sector-overview` | Analyze industry: peer comparisons, margin trends, growth drivers | Comparative analysis across peer group; highlights relative value |
| `idea-generation` | Analyze stock screener output: which stocks are most attractive | Qualitative screen interpretation; links to valuation framework |
| `morning-note` | Daily summary: overnight news, market setup, key research updates | Aggregates news, internal alerts, market data |

## Hooks

**File:** `/equity-research/hooks/hooks.json`

Hooks ที่มี:
- **Post-earnings release**: Auto-trigger earnings-analysis skill, flag material surprises
- **Weekly**: Auto-generate morning-note summary
- **Catalyst date approaching**: Send reminder to research team

## MCP Connectors ที่เกี่ยวข้อง

| Connector | ใช้สำหรับ |
|-----------|---------|
| **Earnings Calendar** | Earnings dates, release times, consensus estimates |
| **SEC EDGAR** | 10-K, 10-Q, 8-K filings, MD&A |
| **Market Data** | Stock prices, beta, treasury rates (WACC inputs) |
| **Consensus Data** | Analyst estimates, price targets |
| **News Feed** | Company news, sector news, regulatory updates |

## ความสัมพันธ์กับ Agent Plugins

**Equity Research** consumes from **Financial Analysis**:
- `/initiate` ใช้ `dcf-model`, `3-statement-model`, `comps-analysis`
- `/model-update` ใช้ `3-statement-model`
- `/earnings-preview` ใช้ simplified DCF

**Provides to other agents:**
- `portfolio-monitor-agent` — monitor thesis breaches
- `sector-analyst-agent` — sector overview templates

## ตัวอย่างการใช้งาน

### Workflow: Initiating Coverage (10 hours)

```
TASK 1: Research (2 hours)
- Analyst loads /initiate Tesla
- Plugin gathers: business model (EVs, energy, software), 
  competitive set (legacy automakers, EV-native), 
  market position (EV leader but growing competition)

TASK 2: Model (3 hours)
- Plugin builds 10-year model:
  Base case: EV unit growth 12% CAGR, Margin expansion to 20% by Y10
  Bear case: Competition pressure, margin caps at 15%
  Bull case: FSD monetization, energy upside, margin 25%
- WACC: 8.5% (cost of equity 10% given beta 1.8)
- DCF valuation: $280 (bear), $410 (base), $580 (bull)

TASK 3: Thesis (1 hour)
BULL CASE (30% weight):
- EV adoption accelerating (inflection 2026-2027)
- FSD monetization adds $100B+ TAM (autonomous mobility)
- Energy business (grid-scale storage) growing 40% CAGR
Key metrics to monitor: EV margin %, FSD adoption rate

BEAR CASE (20% weight):
- Legacy OEMs catch up, EV margins compress to 5%
- FSD never achieves full autonomy, monetization fails
- Energy remains <5% of revenue
Key metrics to monitor: Competitive EV pricing, legal/regulatory FSD risks

NEUTRAL CASE (50% weight):
- Steady EV growth, steady margin, modest FSD gains
- Price target: $410 (base case)

TASK 4: Report (3 hours)
- Plugin writes 20-page report:
  * Investment Thesis (1 page)
  * Company Overview (2 pages)
  * Industry Analysis (2 pages)
  * Competitive Positioning (2 pages)
  * Financial Model (3 pages)
  * DCF Valuation (2 pages)
  * Risks (2 pages)
  * Rating & Price Target (1 page)

TASK 5: Publish
- Report ready for compliance review + distribution
```

### Workflow: Earnings Update (2 hours)

```
ACTUAL RESULTS (Tuesday 4 PM):
Revenue: $27.3B (consensus $26.8B, model $27.0B)
EPS: $1.15 (consensus $0.98, model $1.10)
Guidance: FY26 revenue $320-330B (our model $325B — ON TARGET)
EBITDA margin: 24% (above our 22% assumption)

Plugin loads /earnings
1. Compares line-by-line vs model:
   ✓ Revenue beat: +0.3% vs model, +1.9% vs consensus
   ✓ EPS beat: +0.05 vs model, +17.3% vs consensus (margin upside)
   ✓ Guidance in line with our 2026 forecast

2. Identifies drivers:
   - Pricing power better than expected (+200 bps margin)
   - Operating leverage from scale (R&D deleverage)
   - Guidance confident (no downside scenario)

3. Updates thesis:
   - Bull case probability up from 30% → 35%
   - Model margin assumption up 1-2 bps
   - Price target: $410 → $425 (modest increase)

4. Actions:
   - Update model with new guidance
   - Alert portfolio managers (if overweight)
   - Schedule management call
```

### Workflow: Morning Note (30 min)

```
OVERNIGHT:
- Fed Chair spoke (Treasury yields up 10 bps)
- Amazon announced new GPU AI chips (NVIDIA loses TAM)
- Tech sector down 1.5% in pre-market

Plugin loads /morning-note
1. Gathers overnight news across coverage universe
2. Highlights NVIDIA impact: competitive AI threat
3. Updates Tesla catalyst calendar: Model Y refresh pushed to 2026 Q3
4. Output: 1-page note with:
   - Market setup (yields, sector momentum)
   - Key stories (Fed policy, competitive AI, EV catalysts)
   - Stock impact (NVIDIA -3%, TSLA -1%)
   - Research outlook (no changes to calls)
```

## Files

| Path | ทำอะไร |
|------|---------|
| `commands/initiate.md` | /initiate coverage workflow |
| `commands/earnings.md` | /earnings analysis workflow |
| `commands/earnings-preview.md` | /earnings-preview workflow |
| `commands/model-update.md` | /model-update workflow |
| `commands/catalysts.md` | /catalysts calendar workflow |
| `commands/thesis.md` | /thesis documentation workflow |
| `commands/morning-note.md` | /morning-note workflow |
| `commands/sector.md` | /sector analysis workflow |
| `commands/screen.md` | /screen analysis workflow |
| `skills/initiating-coverage/SKILL.md` | 5-step initiation |
| `skills/earnings-analysis/SKILL.md` | Earnings comparison |
| `skills/earnings-preview/SKILL.md` | Earnings modeling |
| `skills/model-update/SKILL.md` | Model refresh |
| `skills/thesis-tracker/SKILL.md` | Thesis management |
| `skills/catalyst-calendar/SKILL.md` | Event tracking |
| `skills/sector-overview/SKILL.md` | Peer comparison |
| `skills/idea-generation/SKILL.md` | Screen analysis |
| `skills/morning-note/SKILL.md` | Daily summary |
| `.claude-plugin/plugin.json` | Plugin metadata |

---

**Research Vertical = Report factory**: Coverage initiation, earnings tracking, model updates, thesis monitoring
