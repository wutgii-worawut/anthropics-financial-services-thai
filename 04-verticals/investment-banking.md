---
title: Investment Banking
parent: 04 Vertical Plugins
nav_order: 2
---

# Investment Banking — ศูนย์กลาง M&A

**Investment Banking** vertical ช่วย M&A teams, corporate development professionals, และ sell-side advisors จัดการ deal workflows — จาก teaser, CIM, pitch deck, จนถึง merger modeling แล้ว deal tracking

## ภาพรวม

เป้าหมาย: Accelerate M&A deal execution — คน เอกสาร การวิเคราะห์

**ผู้ใช้:**
- Investment bankers (M&A advisory, sell-side, buy-side)
- Corporate development
- Transaction managers
- Deal teams (senior bankers, associates, analysts)

**Workflows ที่สนับสนุน:**
- Sell-side process (teaser → CIM → datapack → buyer list)
- Buy-side due diligence (letter review, profile extraction, management team research)
- Merger modeling (synergies, accretion/dilution analysis)
- Pitch deck creation (one-pager, merger model presentation)
- Deal tracking (timeline, status, buyer interest)

## Commands (Slash Commands)

| คำสั่ง | คำอธิบาย |
|---------|---------|
| `/teaser` | สร้าง one-page anonymous teaser สำหรับ sell-side process (หน้าแรก deal) |
| `/cim` | สร้าง Confidential Information Memorandum (CIM) — 50+ page flip book สำหรับ serious buyers |
| `/datapack` | อัดรวม deal data: financials, comps, DCF, market research เป็น single pack |
| `/process-letter` | วิเคราะห์ buyer letter of intent (LOI) — ชี้ term ที่สำคัญ, flag risks |
| `/buyer-list` | สร้าง target buyer list จาก sector research + M&A history |
| `/deal-tracker` | ติดตาม deal timeline, buyer progress, key milestones |
| `/one-pager` | สร้าง executive summary (1 page) สำหรับ quick overview |
| `/merger-model` | สร้าง merger model: pro forma, synergies, accretion/(dilution) |

### ตัวอย่างการใช้ Commands

**Teaser:**
```
/teaser
Input: Company description, $50M revenue, 25% EBITDA margin, Northeast US
Output: One-page PDF, anonymized, "Project [Codename]"
```

**CIM:**
```
/cim ABC Manufacturing
Output: 60-page PDF flip book with company overview, financials, 
market opportunity, management team, risk factors, investment highlights
```

**Merger Model:**
```
/merger-model XYZ acquired by BUYER at $100M purchase price
Assumptions: $8M EBITDA, 4.0x purchase multiple, 30% cost synergies
Output: Pro forma combined financials, accretion to buyer, payback period
```

## Skills ที่มี

| ชื่อ Skill | ทำอะไร | วิธีหลัก |
|-----------|---------|---------|
| `teaser` | 1-page anonymous profile — highlight investment thesis, financial summary, anonymization check | Structured template; sector descriptor; investment highlights; financial table |
| `cim-builder` | Multi-section CIM: executive summary, company overview, market, financials, mgmt team, risks, opportunity | Template-driven; sections auto-populated from data; compliance review |
| `datapack-builder` | Bundle financials, comps, DCF, market research into single deal pack | Combines outputs from financial-analysis vertical |
| `pitch-deck` | PowerPoint presentation: merger model, synergy waterfall, accretion bridge | python-pptx; charts, slides, narrative flow |
| `deal-tracker` | Project timeline, buyer list status, key date reminders | Database-driven; integrates with email/calendar |
| `process-letter` | Parse LOI/offer letter — extract key terms (price, earnout, reps/warranties, closing conditions) | NLP-based term extraction; compliance flags |
| `strip-profile` | Extract management team, org structure, key employee data | Data extraction from CIM, LinkedIn, company websites |
| `buyer-list` | Generate target buyer list based on sector, size, strategy | M&A database research + historical acquirers |
| `merger-model` | Pro forma combined financials; synergy waterfall; accretion analysis | Linked to financial-analysis DCF/3-statement models |

## Hooks

**File:** `/investment-banking/hooks/hooks.json`

Hooks ที่มี:
- **Post-CIM generation**: Auto-notify deal team, flag for compliance review
- **Post-datapack**: Auto-format for distribution, track recipient list

## MCP Connectors ที่เกี่ยวข้อง

| Connector | ใช้สำหรับ |
|-----------|---------|
| **Gmail** | Track correspondence with buyers, advisors, counsel |
| **Slack** | Internal deal notifications, team updates |
| **LinkedIn** | Management team research, buyer profiling |
| **D&B / Crunchbase** | Company data, sector trends |

## ความสัมพันธ์กับ Agent Plugins

**Investment Banking** consumes skill จาก **Financial Analysis**:
- `/cim` ใช้ `dcf-model`, `comps-analysis` สำหรับ valuation section
- `/merger-model` ใช้ `3-statement-model`, `lbo-model`
- `/datapack` combine ทั้ง DCF + comps + market data

**Provides inputs to Agent Plugins:**
- `pitch-agent` — ใช้ pitch deck templates สำหรับ buyer presentations
- `due-diligence-agent` — ใช้ CIM structure เป็น template

## ตัวอย่างการใช้งาน

### Workflow: Sell-Side M&A Process (8 weeks)

```
WEEK 1-2: Prepare Process

1. Advisor loads /teaser
   → Collects company info, financials
   → Creates anonymous 1-pager
   → Output: PDF teaser + tracking spreadsheet

2. Advisor loads /buyer-list ABC Inc.
   → Searches M&A database, recent deals in sector
   → Identifies 15-20 potential strategic + financial buyers
   → Output: Buyer list with contact info, rationale

WEEK 3-4: CIM & Datapack

3. Advisor loads /cim
   → Gathers 3Y historical financials, projections
   → Collects management team bios, org chart
   → Gets competitive analysis, market data
   → Creates 60-page flip book with investment highlights
   → Output: PDF CIM + internal template

4. Advisor loads /datapack
   → Combines financial-analysis outputs (DCF, comps)
   → Adds management strip, buyer synergy analysis
   → Output: Comprehensive deal pack (Excel + PDF)

WEEK 5-6: Marketing Phase

5. Advisor sends teasers to buyer list
   → Tracks opens, interest signals
   → Can update /deal-tracker with buyer status

WEEK 7-8: LOI & Negotiation

6. Advisor receives buyer LOI
   → Loads /process-letter to parse key terms
   → Identifies missing reps, earnout triggers, closing conditions
   → Flags legal/commercial risks

7. Advisor loads /merger-model with buyer's proposed terms
   → Models buyer's acquisition cost
   → Calculates synergies (cost, revenue, tax)
   → Output: Pro forma combined company financials
```

### Workflow: Buyer-Side Merger Model

```
1. Corporate development (buyer side) loads /merger-model
   Target: $50M EBITDA company at proposed 6.0x multiple
   
2. Inputs:
   - Standalone target: $300M revenue, $50M EBITDA, growing 10%
   - Buyer standalone: $2B revenue, $300M EBITDA
   - Synergies: $5M cost savings year 1, $8M by year 3
   
3. Output:
   Pro forma combined:
   - Year 1: $2.3B revenue, $353M EBITDA (11% EBITDA margin)
   - Cost synergies accrete $0.15 per share year 1
   - Full run-rate $0.22 per share accretion by year 3
   
   Sensitivity: Model shows accretion persists unless synergy
   realization drops below 40% — acceptable risk profile
```

## Files

| Path | ทำอะไร |
|------|---------|
| `commands/teaser.md` | /teaser workflow — 1-page profile creation |
| `commands/cim.md` | /cim workflow — full CIM generation |
| `commands/datapack.md` | /datapack workflow — comprehensive deal pack |
| `commands/process-letter.md` | /process-letter workflow — LOI parsing |
| `commands/buyer-list.md` | /buyer-list workflow — target identification |
| `commands/deal-tracker.md` | /deal-tracker workflow — timeline management |
| `commands/one-pager.md` | /one-pager workflow |
| `commands/merger-model.md` | /merger-model workflow |
| `skills/teaser/SKILL.md` | Teaser builder |
| `skills/cim-builder/SKILL.md` | CIM generation |
| `skills/datapack-builder/SKILL.md` | Data bundling |
| `skills/pitch-deck/SKILL.md` | Presentation deck creation |
| `skills/deal-tracker/SKILL.md` | Deal management |
| `skills/process-letter/SKILL.md` | LOI analysis |
| `skills/buyer-list/SKILL.md` | Buyer targeting |
| `skills/merger-model/SKILL.md` | Merger modeling |
| `skills/strip-profile/SKILL.md` | Management extraction |
| `.claude-plugin/plugin.json` | Plugin metadata |

---

**IB Vertical = Deal machinery**: Teaser → CIM → Merger Model → อัปเดต Deal Tracker
