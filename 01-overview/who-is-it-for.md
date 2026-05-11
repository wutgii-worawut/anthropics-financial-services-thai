---
title: ใครเป็นเป้าหมาย
parent: 01 ภาพรวม
nav_order: 2
---

# ใครเป็นเป้าหมาย

Claude for Financial Services สร้างมาสำหรับ **สถาบันการเงิน (FI) และทีม AI ภายใน** ที่ต้องการ reference implementation ของ Claude agents สำหรับวิธีการทำงาน FSI

## ใครได้ประโยชน์สูงสุด

### 1. ทีม Investment Banking & M&A Advisory
- **MD / Director-level**: ต้องการ pitch deck ฉบับร่างด่วน (Pitch Agent, Meeting Prep Agent)
- **Senior Analysts**: ต้องการ comps analysis, precedent transaction database, football field valuations ที่ทำอย่างรวดเร็ว (/comps, /precedents, /dcf commands)
- **Associate cohorts**: ต้องการ automation สำหรับ CIM drafting, teaser generation, buyer list creation (investment-banking vertical)

ยกตัวอย่าง: การสร้าง pitch deck สำหรับ M&A situation analysis ที่ปกติใช้เวลา 2–3 วัน สามารถลดเหลือ 4–6 ชั่วโมง (Pitch Agent agents/pitch-agent.md)

### 2. Equity Research & Coverage Teams
- **Equity analysts**: ต้องการ post-earnings analysis, model updates, initiation reports ที่ต่างหาก (/earnings, /model-update, /initiate commands)
- **Sector heads**: ต้องการ competitive landscape mapping, peer comp freshness (/sector, /screen commands)
- **Publishing teams**: ต้องการ draft consistency & compliance checking ผลลัพธ์ก่อน distribute

ยกตัวอย่าง: Market Researcher agent สามารถสร้าง industry briefing pack พร้อม peer set comps และ idea shortlist จากการ prompt หนึ่งบรรทัด (market-researcher agent agents/market-researcher.md)

### 3. Private Equity & Growth Equity
- **Sourcing teams**: ต้องการ deal screen acceleration, founder outreach draft, CRM lookup (/source, /screen-deal commands)
- **Investment committee**: ต้องการ IC memo drafting, unit economics analysis, returns sensitivity tables (/ic-memo, /unit-economics, /returns commands)
- **Portfolio operations**: ต้องการ KPI tracking, variance analysis, value-creation-plan updates (/portfolio, /value-creation commands)
- **Due diligence**: ต้องการ management interview prep, checklist generation (/dd-prep, /dd-checklist commands)

ยกตัวอย่าง: KYC Screener agent ทำการ parse onboarding documents และ run rules engine เพื่อหา gaps — ลดเวลา compliance screening จากหลายชั่วโมงเป็นนาที (kyc-screener agent agents/kyc-screener.md, operations vertical)

### 4. Fund Administration & Finance Ops
- **Accounting teams**: ต้องการ GL reconciliation automation, break tracing, month-end close support (GL Reconciler, Month-End Closer, Statement Auditor agents)
- **Controllers**: ต้องการ variance commentary draft, accrual tracking, roll-forward verification (fund-admin vertical)
- **LP relations**: ต้องการ NAV tie-out support, statement audit, valuation review staging (/valuation-reviewer command)

ยกตัวอย่าง: GL Reconciler agent อ่าน trial balance, query subledger ledgers ผ่าน MCP, หา discrepancies และ trace root cause — ลดเวลา manual recon 1–2 สัปดาห์ (managed-agent-cookbooks/gl-reconciler README)

### 5. Wealth Management & Financial Planning
- **Client advisors**: ต้องการ financial plan generation, portfolio review templates, rebalancing recommendations (/financial-plan, /client-review, /rebalance commands)
- **Planning specialists**: ต้องการ retirement projections, education funding scenarios, estate tax optimization (/financial-plan command)
- **Operations**: ต้องการ tax-loss harvesting opportunity identification, client reporting automation (/tlh, /client-report commands)

ยกตัวอย่าง: Wealth Management vertical มี tax-loss harvesting skill ที่ screen portfolio เพื่อหา TLH opportunities พร้อม wash sale management (wealth-management vertical)

## ใครสามารถ Fork & Customize

สถาบันการเงินใดๆ ที่ต้องการจะ fork repository นี้เพื่อ:

- **เพิ่มข้อมูล internal context**: ใส่ firm terminology, legal templates, compliance rules ลงใน skill files
- **เชื่อมต่อ MCP connectors ของตัวเอง**: แทนที่ .mcp.json เพื่อ point ไปยัง internal data providers หรือ vendor APIs ของ firm (README.md line 154–155)
- **ปรับแต่ง agents สำหรับ workflow จริง**: แก้ไข `agents/<slug>.md` system prompt เพื่อให้ match exact process ของทีมตัวเอง
- **เพิ่ม agents ใหม่**: copy structure ของ agent ที่มีอยู่และ customize สำหรับ workflow ที่ไม่ได้ cover (README.md line 158)

### ต้องมี
- Claude API key + Anthropic account (สำหรับ API access)
- MCP provider credentials (ตัวเลือก — ขึ้นอยู่กับ connectors ที่เปิดใช้)
- Internal LLM gateway หรือ Bedrock endpoint (สำหรับ self-hosted deployment)
- AI/ML engineering team ขนาดเล็ก (2–4 คน) เพื่อ maintain custom agents

## ใครไม่ใช่กลุ่มเป้าหมาย

### ❌ Retail Traders & Individual Investors
Repository นี้ไม่ได้ออกแบบมาสำหรับ retail trading หรือ personal investment decisions

- Agents มี compliance guardrails ที่ถูกเขียนสำหรับ institutional users เท่านั้น
- Agents ไม่ execute trades, approve onboarding หรือ bind risk — ทำงาน draft work product สำหรับ review โดย professionals
- Skill ใช้ institutional data providers (FactSet, CapIQ, LSEG) ที่ retail users ไม่มี access

### ❌ Financial Advisors (Individual Contributors)
FA individuals ไม่ใช่ use case หลัก แม้ว่า wealth-management vertical อาจจะมีประโยชน์

- Institutional setup ต้อง Anthropic API key + managed agent deployment — FAs ส่วนใหญ่ใช้ Cowork web UI แทน
- Compliance rules อยู่ที่สถาบันระดับ, ไม่ใช่ individual practitioner

### ❌ "Financial Advice" User Cases
Repository ไม่ให้ investment advice เลย

> ไม่มีสิ่งที่อยู่ใน repository นี้ที่ประกอบด้วย investment, legal, tax หรือ accounting advice
> 
> (README.md line 7–8)

Agents draft work product สำหรับ review โดยผู้เชี่ยวชาญที่มีคุณสมบัติ ผู้ใช้มีความรับผิดชอบในการ verify outputs

## ตัวอย่างหน่วยงานที่อยู่ใน Scope

### ✅ Investment Banks
- Goldman Sachs, Morgan Stanley, JPMorgan, Lazard, Evercore
- **ใช้ Pitch Agent + Meeting Prep Agent**: draft pitch books, comps, LBO models
- **ใช้ investment-banking vertical**: CIM drafting, teaser generation, process letters

### ✅ Equity Research Shops
- Independent research firms, sell-side equity teams, research aggregators
- **ใช้ Market Researcher agent**: sector briefings, competitive analysis
- **ใช้ Earnings Reviewer agent**: post-earnings note drafting
- **ใช้ equity-research vertical**: earnings previews, initiations, thesis tracking

### ✅ PE Firms
- Lower mid-market (50M–250M AUM), growth equity, family offices
- **ใช้ KYC Screener**: deal sourcing qualification
- **ใช้ private-equity vertical**: DD checklists, IC memos, portfolio monitoring, value-creation planning
- **ใช้ Model Builder agent**: LBO model generation, returns analysis

### ✅ Fund Administrators & Operations
- Institutional fund administrators, in-house ops teams at PE/VC firms
- **ใช้ GL Reconciler agent**: GL break tracing
- **ใช้ Month-End Closer agent**: accrual variance commentary
- **ใช้ Statement Auditor agent**: LP statement audit
- **ใช้ fund-admin vertical**: NAV tie-out, valuation review, reporting support

### ✅ Wealth Management Firms
- RIAs, boutique wealth practices, family office operations
- **ใช้ wealth-management vertical**: financial planning, portfolio rebalancing, tax-loss harvesting, client reporting

---

## ดูเพิ่มเติม

- [What is it?](/01-overview/what-is-it.md) — องค์ประกอบหลักและวิธีการ dual runtime
- [Architecture](/01-overview/architecture.md) — ลำดับชั้น 3-layer: verticals → agents → cookbooks
- [Quick Start](/01-overview/quickstart.md) — ติดตั้ง Cowork หรือ Managed Agents API
