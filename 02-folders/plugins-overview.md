---
title: Plugins Overview — การจัดหมวดหมู่ 3 ทิศทาง
parent: 02 โครงสร้างโฟลเดอร์
nav_order: 4
---

# Plugins Directory Structure

ที่อยู่: `plugins/`

Plugins ถูกแบ่งออกเป็น **3 categories** ตามวัตถุประสงค์และการจัดการ

```
plugins/
├── agent-plugins/      # 10 ตัวแทนชื่อ (named agents)
├── vertical-plugins/   # 8 vertical + core ทักษะ
└── partner-built/      # Partner plugins (LSEG, S&P Global)
```

## 1. agent-plugins/ — ตัวแทนชื่อ (10 agents)

### ลักษณะเฉพาะ

**ตัวแทนจบสำเร็จ** — ตั้งแต่ input ถึง output ที่ใช้งานได้เลย
- ชื่อที่ชัดเจน: `pitch-agent`, `market-researcher`, `gl-reconciler`
- Self-contained: ทุก agent bundles ทักษะ (skills) ที่ใช้ตัวเอง
- ดำเนินการ workflow ที่ชัดเจน (comps → pitch deck, earnings → model update, ...)

### โครงสร้างแต่ละ agent

```
agent-plugins/pitch-agent/
├── .claude-plugin/
│   └── plugin.json         # metadata
├── agents/
│   └── pitch-agent.md      # canonical system prompt
└── skills/
    ├── comps-analysis/     # bundled copy
    ├── precedent-analysis/
    ├── lbo-model/
    └── deck-author/        # write PowerPoint
```

### 10 Agents

| Agent | Vertical Source | Workflow |
|---|---|---|
| `pitch-agent` | investment-banking | Comps + precedents + LBO → branded pitch deck |
| `market-researcher` | equity-research | Sector/theme → overview, landscape, peer comps, ideas |
| `earnings-reviewer` | equity-research | Earnings call + filings → model update → note |
| `meeting-prep-agent` | wealth-management | Client ID → briefing pack |
| `model-builder` | financial-analysis | Ticker + assumptions → DCF/LBO/3-stmt Excel file |
| `gl-reconciler` | financial-analysis | GL vs subledger → break detection + root cause |
| `kyc-screener` | financial-analysis | Onboarding docs → rules check + gap flags |
| `valuation-reviewer` | private-equity | GP packages → valuation template → LP reporting |
| `month-end-closer` | financial-analysis | Entity + period → accruals + roll-forwards + commentary |
| `statement-auditor` | private-equity | LP statements → NAV tie-out + audit flags |

### คุณสมบัติ

✓ ติดตั้งผ่าน Cowork (1 plugin = 1 agent)
✓ Deploy ผ่าน Managed Agents API (matching cookbook ใน `managed-agent-cookbooks/`)
✓ Reusable skills → bundled copies

## 2. vertical-plugins/ — แนวตั้งและทักษะ (8 + core)

### ลักษณะเฉพาะ

**ห้องสมุดทักษะที่จัดกลุ่มตามปัญหา** — ไม่จบสิ้น แต่ให้ basic toolkit
- มี skills, commands, MCP connectors ต่อ domain
- **source of truth** สำหรับทักษะ (agents copy ไปยัง bundles)
- ไม่ได้มี system prompt (agents เพิ่มอันเป็นของตัวเอง)

### โครงสร้างแต่ละ vertical

```
vertical-plugins/financial-analysis/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   ├── comps.md           # /comps command
│   ├── dcf.md             # /dcf command
│   └── ...
├── skills/
│   ├── comps-analysis/    # source of truth
│   ├── dcf-model/
│   ├── lbo-model/
│   └── ...
└── .mcp.json              # data connector definitions
```

### 9 Verticals

| Vertical | Purpose | Main Skills | MCP Connectors |
|---|---|---|---|
| **financial-analysis** *(core)* | Modeling, Excel, deck QC | comps, dcf, lbo, 3-statement, audit-xls, deck-refresh | Daloopa, Morningstar, FactSet, S&P, Moody's, MT News, Aiera, Chronograph |
| **investment-banking** | Deal materials | pitch-deck, cim-builder, teaser, buyer-list, merger-model, process-letter, deal-tracker | — |
| **equity-research** | Coverage & publishing | earnings-analysis, earnings-preview, initiating-coverage, model-update, thesis-tracker, catalyst-calendar | — |
| **private-equity** | Sourcing & portfolio | deal-sourcing, dd-checklist, ic-memo, returns-analysis, portfolio-monitoring | — |
| **wealth-management** | Advisor workflows | client-review, financial-plan, portfolio-rebalance, client-report, tax-loss-harvesting | — |
| **fund-admin** | Finance ops | (shared with financial-analysis, extends it) | — |
| **operations** | Compliance | kyc-document-parsing, rules-grid-evaluation | — |
| **lseg** *(partner)* | Bond analytics (LSEG data) | bond-rv, swap-curves, fx-carry, options-vol | LSEG MCP |
| **sp-global** *(partner)* | S&P Capital IQ | tear-sheets, earnings-previews, funding-digests | S&P Global MCP |

### คุณสมบัติ

✓ ติดตั้งแยกต่างหาก (ผู้ใช้เลือกเฉพาะ verticals ที่ต้องการ)
✓ Agents copy skills ตรงนี้ → bundled copies (ดู `sync-agent-skills.py`)
✓ MCP connectors (financial-analysis มีทั้ง 11)

## 3. partner-built/ — Plugins จากพันธมิตร

### ลักษณะเฉพาะ

**เครื่องมือจากคู่ค้า** — ต้องมี subscription/license จากผู้ให้บริการ
- LSEG (Refinitiv) — bond analytics, rates, FX
- S&P Global — Capital IQ data (tear sheets, earnings)

### โครงสร้าง

```
partner-built/lseg/
├── .claude-plugin/plugin.json
├── commands/
├── skills/
└── .mcp.json                 # points to LSEG's MCP server

partner-built/spglobal/
├── .claude-plugin/plugin.json
├── commands/
├── skills/
└── .mcp.json                 # points to S&P Global's MCP server
```

### คุณสมบัติ

✓ ติดตั้งเหมือน vertical plugins
✓ Requires: subscription + API credentials
✓ MCP servers hosted by partner (external)

## ความสัมพันธ์ระหว่าง 3 categories

```
────────────────────────────────────────────────────────────────

vertical-plugins/ (source of truth)
├── financial-analysis
│   ├── skills/comps-analysis/  ← shared library
│   ├── skills/dcf-model/
│   └── .mcp.json               ← connectors (Daloopa, etc.)
├── investment-banking
│   ├── skills/cim-builder/     ← shared library
│   └── ...
└── equity-research
    ├── skills/earnings-analysis/  ← shared library
    └── ...

                 ⬇️ (sync-agent-skills.py)
                 
agent-plugins/ (bundled copies)
├── pitch-agent/
│   ├── agents/pitch-agent.md        ← own system prompt
│   ├── skills/comps-analysis/       ← copy (from vertical)
│   ├── skills/precedent-analysis/   ← copy (from vertical)
│   └── skills/lbo-model/            ← copy (from vertical)
├── market-researcher/
│   ├── agents/market-researcher.md
│   └── skills/...                   ← copies (from equity-research vertical)
└── earnings-reviewer/
    ├── agents/earnings-reviewer.md
    └── skills/...                   ← copies (from equity-research vertical)

────────────────────────────────────────────────────────────────

partner-built/ (external)
├── lseg/
│   ├── .mcp.json  → https://api.analytics.lseg.com/lfa/mcp
│   └── skills/    (LSEG's domain-specific skills)
└── spglobal/
    ├── .mcp.json  → https://kfinance.kensho.com/integrations/mcp
    └── skills/    (S&P's domain-specific skills)
```

## เหตุที่แบ่ง 3 ทิศทาง

| หมวด | ความสาม | ประเด็น | ตัวอย่าง |
|---|---|---|---|
| **agent-plugins** | Self-contained workflow agents | ผู้ใช้ติดตั้ง 1 agent → ได้ workflow เสร็จสิ้น | pitch-agent: ผู้ใช้คนเดียว |
| **vertical-plugins** | Shared skill libraries + connectors | Developers แก้ไขทักษะ 1 ที่ → sync ไปยัง agents ทั้งหมด | comps-analysis: ใช้ใน pitch-agent, market-researcher, model-builder |
| **partner-built** | External, branded tools | Partners control code, API keys, updates | LSEG requires subscription, S&P requires Capital IQ access |

## Sync Mechanism: sync-agent-skills.py

เมื่อ developer แก้ไข skill ใน `vertical-plugins/financial-analysis/skills/comps-analysis/`:

```bash
$ python3 scripts/sync-agent-skills.py
```

Script:
1. Scans `vertical-plugins/*/skills/` ทั้งหมด
2. For each bundled copy ใน `agent-plugins/*/skills/`:
   - Checks ชื่อ skill ที่ตรงกัน
   - ลบ old bundled copy
   - Copy ใหม่จากแหล่ง

**ผล**: ทุก agents ที่ใช้ `comps-analysis` ได้ update อัตโนมัติ

---

**สรุป**: 
- **agent-plugins** = ตัวแทนชื่อสำหรับผู้ใช้ปลายน้อย
- **vertical-plugins** = ห้องสมุดทักษะและ MCP connectors
- **partner-built** = tools ของคู่ค้า (LSEG, S&P)
- ทั้ง 3 ตั้งอยู่ใน plugins/ และ Cowork ค้นหาทั้งหมด ผ่าน marketplace.json
