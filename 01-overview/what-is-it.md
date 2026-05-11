---
title: Claude for Financial Services คืออะไร
parent: 01 ภาพรวม
nav_order: 1
---

# Claude for Financial Services คืออะไร

Claude for Financial Services เป็น **reference implementation ระดับ production** ที่ Anthropic ออกแบบมาสำหรับวิธีการทำงานด้าน financial services ที่พบมากที่สุด — investment banking, equity research, private equity, wealth management และ fund operations

เป็นระบบอวน (open-source repository) ที่สามารถเรียกใช้ได้ **สองวิธีจากแหล่งเดียว**: ติดตั้งเป็น Claude Cowork plugin หรือ deploy เป็น Claude Managed Agents API อยู่หลัง workflow engine ของตัวเอง ระบบ prompt, skills และ conventions เหมือนกันทั้งสองวิธี — เลือกได้ว่าจะรัน ที่ไหน

## องค์ประกอบหลัก

### 10 Named Agents (ตัวแทนชื่อที่รู้จัก)

Repository นี้มี 10 agent ตั้งชื่อเรียบร้อยแล้ว แต่ละตัวเป็น self-contained plugin ที่ครอบคลุมงาน workflow ตั้งแต่ต้นถึงปลาย:

| หมวด | Agent | งานที่ทำ |
|---|---|---|
| **Coverage & advisory** | Pitch Agent | Comps, precedents, LBO → pitch deck ตั้งแต่ต้นถึงปลาย (README.md line 23) |
| | Meeting Prep Agent | เตรียมงาน briefing pack ก่อนทุก client meeting (line 24) |
| **Research & modeling** | Market Researcher | Sector/theme → industry overview, competitive landscape, comps, shortlist ideas (line 25) |
| | Earnings Reviewer | Earnings call + filings → update model → draft note (line 26) |
| | Model Builder | DCF, LBO, 3-statement, comps ขึ้นใน Excel แบบ live (line 27) |
| **Fund admin & ops** | Valuation Reviewer | Ingest GP packages, run valuation template, stage LP reporting (line 28) |
| | GL Reconciler | หา breaks, trace root cause, route for sign-off (line 29) |
| | Month-End Closer | Accruals, roll-forwards, variance commentary (line 30) |
| | Statement Auditor | Audit LP statements ก่อน distribution (line 31) |
| **Operations & onboarding** | KYC Screener | Parse onboarding docs, run rules engine, flag gaps (line 32) |

ทั้ง 10 agent นี้พร้อมใช้งานทันที — แต่เป็นจุดเริ่มต้นสำหรับปรับแต่งตามวิธีทำงานของแต่ละสถาบัน

### 7 Vertical Plugins (มัดทักษะแบ่งตามแนวตั้ง)

Skills, slash commands และ MCP connectors ถูกรวมกลุ่มตามแนวตั้ง FSI (Financial Services Industry). ติดตั้งแล้ว vertical ที่ต้องการ:

- **financial-analysis** *(core)* — Comps, DCF, LBO, 3-statement, deck QC, Excel audit + 11 data connectors
- **investment-banking** — CIMs, teasers, process letters, buyer lists, merger models, deal tracking
- **equity-research** — Earnings notes, initiations, model updates, thesis tracking
- **private-equity** — Sourcing, screening, diligence checklists, IC memos, portfolio monitoring
- **wealth-management** — Client reviews, financial plans, rebalancing, reporting, tax-loss harvesting
- **fund-admin** — GL recon, break tracing, accruals, roll-forwards, variance commentary, NAV tie-out
- **operations** — KYC document parsing, rules-grid evaluation

(CLAUDE.md line 15–20)

### Dual Runtime (รันบน Cowork หรือ Managed Agents)

**โปรแกรม Cowork plugin** (web UI):
```
ติดตั้ง → Settings → Plugins → Add plugin → https://github.com/anthropics/claude-for-financial-services
```
(README.md line 51–55)

**Claude Managed Agents API** (headless):
```bash
export ANTHROPIC_API_KEY=sk-ant-...
scripts/deploy-managed-agent.sh gl-reconciler
```
(README.md line 80–85)

ทั้งสองวิธีอ่านจากแหล่งเดียว: `agents/<slug>.md` ที่ hold system prompt ตัวเดียวกัน และ `skills/` ที่ bunched เหมือนกัน — แตกต่างกันแค่ wrapper (Cowork ใช้ `.claude-plugin/plugin.json`, Managed Agents ใช้ `agent.yaml`)

## Key Innovation: One Source, Two Delivery

ส่วนสำคัญที่สุดของ Claude for Financial Services คือ **ไม่มีการ duplicate** ระหว่างสองวิธีการรัน

- **system prompt** ✅ ไฟล์เดียว (`agents/pitch-agent.md`)
- **skills** ✅ ไฟล์เดียว (`plugins/vertical-plugins/financial-analysis/skills/dcf-model/SKILL.md`)
- **commands** ✅ ไฟล์เดียว (`plugins/vertical-plugins/investment-banking/commands/cim.md`)

ทั้งสอง wrapper (Cowork + Managed Agents) reference ไฟล์เดียวกัน ถ้า update SKILL.md ของ DCF → ได้ผลกับ Cowork และ API deployment พร้อมกัน (CLAUDE.md line 31)

## 11 MCP Connectors (ต่อเชื่อมข้อมูล)

Repository นี้ไม่ได้ fetch จาก web search แต่ใช้ Model Context Protocol (MCP) servers ที่เป็นแบบหนึ่งจุด (single-purpose, allowlisted) ให้เข้าถึง FSI data providers:

- **Daloopa** — Financial metrics & modeling
- **Morningstar** — Fund data & analytics
- **S&P Global (Capital IQ)** — Company research, tearsheets, transactions
- **FactSet** — Alternative data & research
- **Moody's** — Credit analytics
- **MT Newswires** — News feed
- **Aiera** — Earnings & conference calls
- **LSEG** — Bond data, derivatives, FX curves
- **PitchBook** — M&A, VC, fund data
- **Chronograph** — PE portfolio analytics
- **Egnyte** — Document management

(README.md line 123–135)

แต่ละ MCP server ต่อด้วย API key (ตัวเลือก) ที่ installed ใน `financial-analysis` vertical plugin → แชร์ไปยังทุก agent ที่ใช้ vertical นั้น

## ลักษณะอื่นๆ

### Skill Quality — Manifest-Driven

หากแต่ละ skill ได้รวบรวมกรรมวิธี step-by-step ในไฟล์ SKILL.md:

```markdown
---
name: dcf-model
description: Real DCF model...
---

# DCF Model Builder

## Critical Constraints
- Formulas over hardcodes (ต้องเป็น Excel formula ไม่ใช่ hardcoded number)
- Sensitivity tables ต้อง odd dimensions (5×5 center = base case)
- Verify step-by-step with user ไม่ build end-to-end
...
```

ไม่มี unit tests — validation มาจากการ lint manifest, verify references resolve (scripts/check.py) และ drift detection (scripts/sync-agent-skills.py) (CLAUDE.md line 31–32)

### Hard-Allowlist Orchestration

ทุก agent และ skill มีรายชื่อ **exact permissions** ที่ได้อนุญาต — ไม่ trust Claude-to-Claude output

GL Reconciler agent เช่น:

```yaml
tools:
  - type: agent_toolset_20260401
    configs:
      - name: read        # allowed
        enabled: true
      - name: grep        # allowed
        enabled: true
  - type: mcp_toolset
    mcp_server_name: internal-gl
    default_config:
      enabled: true       # read-only GL server
```

(managed-agent-cookbooks/gl-reconciler/agent.yaml line 16–33)

ไม่ได้ allow write, bash execution หรือ network outbound นอกจากที่ explicit allowlist

### Vertical-First Structure

Skill source ของ truth ตั้งอยู่ใน `plugins/vertical-plugins/<vertical>/skills/` แล้ว propagate เข้า agent bundles ผ่าน `sync-agent-skills.py` ถ้า edit DCF skill → ต้อง run script เพื่อ update ทุก agent ที่เป็นการ bundle นั้น

(README.md line 95–96, CLAUDE.md line 31)

---

## คำเตือน ⚠️

> ไม่มีสิ่งที่อยู่ใน repository นี้ที่ประกอบด้วย investment, legal, tax หรือ accounting advice

Agents นี้ draft analyst work product — models, memos, research notes, reconciliations — สำหรับ review โดยผู้เชี่ยวชาญที่มีคุณสมบัติ ไม่ได้ make investment recommendations, execute transactions, bind risk, post to ledger หรือ approve onboarding

ผู้ใช้มีความรับผิดชอบในการ verify outputs และ compliance กับกฎหมายและอากฤษยา

(README.md line 7–8)

---

## ดูเพิ่มเติม

- [Who Is It For?](/01-overview/who-is-it-for.md) — เป้าหมายกลุ่มผู้ใช้ประเภทใด
- [Architecture](/01-overview/architecture.md) — วิธีการเชื่อมต่อ 3 layers
- [Quick Start](/01-overview/quickstart.md) — ติดตั้งและลองใช้
