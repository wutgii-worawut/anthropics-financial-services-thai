---
title: Financial Analysis (วิเคราะห์การเงิน)
parent: 04 Vertical Plugins
nav_order: 1
---

# Financial Analysis — ทะยานที่ฐาน

**Financial Analysis** คือ vertical หลักของ FSI plugins — รากของต้นไม้ที่ vertical อื่นทั้งหมดสำรพเรียนกิ่ง คำสั่งและ skill ในที่นี้ให้กำลังคำนวณ DCF, comps analysis, LBO modeling, 3-statement projection แล้วบริหารจัดการ Excel models ทั้งหมด ขึ้นจากชั้น foundational models นี้ investment-banking, equity-research, private-equity ทั้งหมด จึงสามารถ build deck, memo, research ได้

## ภาพรวม

เป้าหมาย: ช่วย financial professionals สร้างโมเดลตัวเลข ประเมินมูลค่าบริษัท และตรวจสอบคุณภาพ Excel

**ผู้ใช้:**
- Investment bankers (M&A, corporate finance)
- Equity research analysts
- PE investors (deal evaluation)
- Corporate finance teams
- Financial advisors

**Workflows ที่สนับสนุน:**
- Discounted Cash Flow (DCF) valuation
- Comparable companies analysis (trading comps, M&A comps)
- Leveraged Buyout (LBO) modeling
- 3-statement financial projections
- Competitive positioning analysis
- PowerPoint deck QC และ refresh
- Excel data cleanup และ validation

## Commands (Slash Commands)

| คำสั่ง | คำอธิบาย |
|---------|---------|
| `/dcf` | สร้างโมเดล DCF ทั้งหมด เทียบ comps multiples, ออก Bear/Base/Bull cases พร้อม sensitivity tables |
| `/comps` | วิเคราะห์บริษัทเปรียบเทียบ หาแบบแบบ multiples (EV/EBITDA, EV/Revenue, P/E) |
| `/3-statement-model` | สร้างโมเดล projection รายได้, ค่าใช้, กำไร 5 ปี |
| `/lbo` | คำนวณ LBO structure: leverage levels, returns to sponsor, exit scenarios |
| `/competitive-analysis` | วิเคราะห์ landscape ตลาด, market share, positioning |
| `/debug-model` | ตรวจหา error ใน Excel formulas, ชี้ #REF!, #DIV/0!, etc. |
| `/ppt-template` | สร้าง template PowerPoint สำหรับ financial modeling |

### ตัวอย่างการใช้ Commands

**DCF:**
```
/dcf Apple Inc.
```
Plugin จะไป gather historical financials, project 5 years, calculate WACC, 
build Bear/Base/Bull scenarios, สร้าง sensitivity tables — ออกมา Excel file พร้อมใช้

**Comps:**
```
/comps Microsoft 2025E revenue growth
```
Searches สำหรับ comparable public companies ใน software/cloud sector, 
ดึง EV, revenue, EBITDA, เทียบหลาย scenarios

**3-Statement Model:**
```
/3-statement-model SaaS startup $10M revenue, 40% growth, 20% EBITDA margin
```
Creates projection ที่ consistent: Income Statement, Balance Sheet, Cash Flow

## Skills ที่มี

### Core Valuation Skills

| ชื่อ Skill | ทำอะไร | วิธีหลัก |
|-----------|---------|---------|
| `dcf-model` | DCF builder — WACC calculation, 5Y projection, terminal value, sensitivity (WACC vs Terminal g, Revenue vs EBIT margin) | Full discounted cash flow methodology; mid-year convention; three scenario blocks (Bear/Base/Bull) with INDEX consolidation formulas |
| `comps-analysis` | Identify 4-6 peer companies, pull multiples (EV/Revenue, EV/EBITDA, P/E) | Market data + web research; statistical summary (median, 25th/75th percentile) of trading multiples |
| `lbo-model` | Leverage structure, debt paydown, sponsor returns | Sources & uses, leverage ratios, exit scenarios with IRR/MoIC |
| `3-statement-model` | Income statement, balance sheet, cash flow projection | Linked financials; revenue → EBIT → NI; CapEx → cash; working capital |

### Excel Authoring & Cleanup Skills

| ชื่อ Skill | ทำอะไร | วิธีหลัก |
|-----------|---------|---------|
| `xlsx-author` | Create/edit Excel from scratch; formulas, formatting, cell comments | openpyxl-based; blue inputs, black formulas, green sheet links; borders; recalc.py |
| `audit-xls` | Find errors, inconsistencies, broken formulas | Scans all cells for #REF!, #VALUE!, #DIV/0!, etc.; checks formula integrity |
| `clean-data-xls` | Normalize data formats, remove duplicates, standardize | Data cleaning pipeline; handles merged cells, inconsistent numbering |

### Deck & Presentation Skills

| ชื่อ Skill | ทำอะไร | วิธีหลัก |
|-----------|---------|---------|
| `deck-refresh` | Update financial charts/tables in PowerPoint based on new model results | Extracts numbers from Excel → updates PPTX charts + tables |
| `pptx-author` | Create/edit PowerPoint slides; text, charts, tables | python-pptx library; layouts, master slides, chart insertion |
| `ppt-template-creator` | Build reusable PowerPoint templates (e.g., "IB Pitch Deck", "Research Initiation") | Template design with placeholder slide layouts |
| `ib-check-deck` | QC PowerPoint deck — check numbers against model, verify formatting, flag inconsistencies | Extraction + cross-check; ensures financial consistency |

### Advanced Skills

| ชื่อ Skill | ทำอะไร | วิธีหลัก |
|-----------|---------|---------|
| `competitive-analysis` | Market positioning, competitor comparison, SWOT, market share | Framework-driven (Porter 5 Forces, SWOT, Value Chain) + research |
| `skill-creator` | Meta-skill: create new skills from templates | Template generation, validation workflow |

## Hooks

**File:** `/financial-analysis/hooks/hooks.json`

จำนวน hooks ในไฟล์ = **0** (ไม่มี pre/post hooks; commands invoke skills directly)

## MCP Connectors ที่เกี่ยวข้อง

| Connector | ใช้สำหรับ |
|-----------|---------|
| **Market Data MCP** (stock prices, beta, treasury rates) | WACC calculation, valuation inputs |
| **SEC EDGAR MCP** | Historical financials, 10-K/10-Q extraction |
| **Daloopa (optional)** | Structured financial data, peer metrics |
| **Web Search/Fetch** | Current news, analyst estimates, industry reports |

## ความสัมพันธ์กับ Agent Plugins

**Financial Analysis** exports skills ไปยังตัวอักษร vertical:

- **Investment Banking**: `/dcf` ใช้ `dcf-model` skill; `/cim` ใช้ `comps-analysis`; `/teaser` บาง metrics ใช้มา
- **Private Equity**: `/screen-deal`, `/dd-prep`, `/ic-memo` ทั้งหมด reference `dcf-model`, `lbo-model`, `3-statement-model`
- **Equity Research**: `/earnings`, `/model-update` ใช้ `3-statement-model`, `dcf-model`
- **Wealth Management**: `/financial-plan`, `/proposal` ใช้ simplified `3-statement-model`

## ตัวอย่างการใช้งาน

### Workflow: Build DCF สำหรับ M&A Advisory

```
1. Analyst loads /dcf Tesla
   → Plugin gathers 10-K data, current market cap, debt, shares

2. Plugin runs comps-analysis:
   → Finds: Ford, GM, BMW, Volkswagen, Hyundai (5 peers)
   → Pulls EV/EBITDA, EV/Revenue multiples
   → Shows median 8.2x EV/EBITDA, range 6.5x-11.3x

3. Plugin builds DCF:
   → Revenue projection: 18% CAGR (base case) vs historical 45% (moderation)
   → EBIT margin: expand from 15% to 22% by year 5
   → WACC: 7.5% (cost of equity 9.5%, cost of debt 4.2%)
   → Terminal growth: 3.0%

4. Creates three scenarios:
   BEAR: 12% revenue growth, 18% terminal margin, 8.5% WACC → $145/share
   BASE: 18% revenue growth, 22% terminal margin, 7.5% WACC → $210/share  
   BULL: 24% revenue growth, 26% terminal margin, 6.5% WACC → $285/share

5. Plugin auto-creates sensitivity tables:
   WACC vs Terminal Growth (5×5 grid)
   Revenue Growth vs EBIT Margin (5×5 grid)
   Beta vs Risk-Free Rate (5×5 grid)
   
   Base case $210 highlighted in center

6. Outputs:
   - Excel DCF model (2 sheets: DCF + WACC)
   - PDF summary with charts
   - PowerPoint deck with valuation bridge
```

### Workflow: Comps Analysis สำหรับ Equity Research

```
1. Analyst loads /comps NVIDIA
   
2. Plugin identifies:
   → Foundry: ASML, QCOM, AMD, INTC (semiconductors)
   → Platform: AMZN, MSFT, GOOGL (cloud)
   → Gaming: ROBLOX, TAKE2 (gaming)

3. Pulls metrics (9-month forward):
   Revenue, EBITDA, FCF
   EV/Revenue, EV/EBITDA, P/E multiples

4. Output: Excel sheet with comparables grid
   NVIDIA vs peers: P/E 35.2x vs peer median 18.5x (premium justified?)
```

## Files

| Path | ทำอะไร |
|------|---------|
| `commands/dcf.md` | /dcf workflow — 5-step DCF building process |
| `commands/comps.md` | /comps workflow — peer analysis |
| `commands/lbo.md` | /lbo workflow — leverage structure |
| `commands/3-statement-model.md` | /3-statement workflow |
| `commands/competitive-analysis.md` | Market analysis workflow |
| `skills/dcf-model/SKILL.md` | DCF builder implementation (1200+ lines) |
| `skills/3-statement-model/SKILL.md` | 3-statement projection |
| `skills/comps-analysis/SKILL.md` | Comparable company extraction |
| `skills/lbo-model/SKILL.md` | LBO structure modeling |
| `skills/xlsx-author/SKILL.md` | Excel creation utility |
| `skills/audit-xls/SKILL.md` | Excel error detection |
| `.claude-plugin/plugin.json` | Plugin metadata |

---

**นี่คือหัวใจหลักของ FSI ecosystem** — skill อื่นๆ ทั้งหมดขึ้นจากความสามารถของ Financial Analysis นี้
