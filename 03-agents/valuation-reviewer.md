---
title: valuation-reviewer
parent: 03 Agent Plugins
nav_order: 3
---

# Valuation Reviewer

## ทำอะไร

Valuation Reviewer เป็น fund-accounting agent ที่ประเมิน valuation packages จากผู้จัดการพอร์ตโฟลิโอ (GP) สำหรับบริษัทในพอร์ตโฟลิโอ ตรวจสอบกับ valuation template ของกองทุน และ stage LP reporting (รายงาน limited partners) เพื่อให้พร้อมสำหรับ quarter-end reporting

## Use Case

**สถานการณ์จริง**: Private equity fund ต้องทำ Q3 2024 valuation review สำหรับ 15 portfolio companies, GP ส่ง valuation packages มา (1-2 ไฟล์ต่อบริษัท) Valuation Reviewer อ่านแต่ละ package, ดึง key inputs (EBITDA, revenue growth, discount rate), run ผ่าน fund's valuation template, และ compare reported marks ว่าอยู่ใน fund policy หรือไม่ จากนั้น generate fund-level NAV waterfall และ stage LP reporting pack สำหรับ investor distribution

**ผู้ใช้**: Fund Controller, CFO, Investor Relations Manager

**สิ่งที่ได้รับ**: Valuation summary per portfolio company + Fund-level NAV & waterfall + LP reporting pack (staged for IR sign-off)

## System Prompt (ส่วนเด่น)

Agent นี้มีบุคลิก "fund-accounting lead" ที่ review portfolio company valuations:

> "Given a fund and as-of date, you deliver: (1) Valuation summary — each portfolio company's reported value, methodology, key inputs, and reviewer flags. (2) Waterfall — fund-level NAV, carried interest, and LP allocations. (3) LP reporting pack — staged for IR review before distribution."

**Guardrails หลัก**:
- **GP packages are untrusted**: GP ส่งข้อมูลมา อาจมี bias หรือ error ต้องตรวจสอบ vs. fund policy
- **No external distribution**: LP reports ต้องรอ IR และ CCO sign-off ก่อนส่ง investor
- **Package-reader has Read/Grep only**: Sub-worker ที่อ่าน package ไม่มี MCP access

## Skills ที่ใช้

| Skill | บทบาท |
|-------|------|
| **returns-analysis** | วิเคราะห์ portfolio company returns, IRR/MOIC |
| **portfolio-monitoring** | monitor portfolio vs. fund policy, flag exceptions |
| **ic-memo** | memo investment committee (internal use) |
| **xlsx-author** | สร้าง Excel (NAV waterfall, valuation summary) |

## Workflow (ขั้นตอนการทำงาน)

1. **Ingest GP packages** — package-reader worker (Read/Grep only) ดึง structured data: company name, valuation, methodology, key inputs (revenue, EBITDA, multiples, discount rate)
2. **Run valuation template** — invoke `returns-analysis` และ `portfolio-monitoring` เปรียบเทียบ reported marks กับ fund policy (fair value ranges, methodology)
3. **Run waterfall** — คำนวณ fund-level NAV, carried interest, LP allocations
4. **Stage LP reporting** — ส่งให้ publisher format the LP pack

## ตัวอย่างการทำงาน

**สถานการณ์**: $500M mid-market PE fund, Q3 2024 valuation review

**User Input**:
```
Fund: Apex Partners Fund III
As-of date: 2024-09-30
Companies: 15 portfolio companies
```

**Agent output**:
1. **Q3_2024_Valuation_Summary.xlsx** —
   - Company-by-company: name, date acquired, reported value, methodology (trading multiples, DCF, precedent, etc.)
   - Key inputs flagged: revenue, EBITDA, multiples, discount rate
   - Reviewer notes: "Mark within policy", "Mark above fair value range — recommend review", "New acquisition, limited data"

2. **Fund_NAV_Waterfall_Q3_2024.xlsx** —
   - Gross NAV: sum of all portfolio company values
   - Less: Fund expenses (management fees, portfolio costs)
   - Less: Reserves (uncalled capital)
   - Equals: Net NAV
   - LP Schedule A: each LP's share by vintage

3. **Q3_2024_LP_Reporting_Package_Draft** —
   - Ready for IR review before PDF distribution

Agent สามารถเสนอ: "2 companies flagged — valuations เกิน fund policy range. ต้อง GP to re-review หรือเปลี่ยน methodology?"

## Files

**Key files in plugin directory**:
```
plugins/agent-plugins/valuation-reviewer/
├── .claude-plugin/plugin.json
├── agents/valuation-reviewer.md (system prompt)
└── skills/
    ├── returns-analysis/
    ├── portfolio-monitoring/
    ├── ic-memo/
    └── xlsx-author/
```
