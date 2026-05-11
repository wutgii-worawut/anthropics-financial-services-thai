---
title: month-end-closer
parent: 03 Agent Plugins
nav_order: 10
---

# Month End Closer

## ทำอะไร

Month End Closer เป็น controller agent ที่ run month-end close process สำหรับ entity และ period ที่กำหนด โดยสร้าง accrual schedules, roll-forward schedules (ยอด ต้น + transactions − reversals = ยอด ปลาย), variance commentary (P&L และ balance sheet flux vs. prior period และ budget) จากนั้น stage close package สำหรับ controller review และ sign-off ใช้สำหรับ period-end close (ไม่ใช่ daily reconciliation)

## Use Case

**สถานการณ์จริง**: Fund accounting team ต้องทำ December 2024 close สำหรับ fund management company entity (ตัวเอง) Month End Closer จะ:
1. ดึง trial balance Dec 31, 2024
2. สร้าง accrual schedules (accrued expenses, revenue accruals, management fee accruals)
3. สร้าง roll-forward schedules (deferred revenue, prepaid expenses, fixed assets)
4. วิเคราะห์ variance: "P&L revenue up 15% YoY — driven by higher management fees + performance fees", "Cash down $2M — mainly due to distribution to LPs"
5. Draft journal entries สำหรับ accruals
6. Stage close package ให้ controller final approval ก่อน post GL

**ผู้ใช้**: Fund Controller, CFO, Accounting Manager

**สิ่งที่ได้รับ**: Accrual schedule + Roll-forward schedules + Variance commentary + Close package (with draft JEs)

## System Prompt (ส่วนเด่น)

Agent นี้มีบุคลิก "controller's right hand":

> "Given an entity and period (YYYY-MM), you deliver: (1) Accrual schedule — each accrual entry with calculation, support reference, and JE draft. (2) Roll-forward schedules — beginning + activity − reversals = ending, tied to GL. (3) Variance commentary — P&L and balance-sheet flux vs. prior period and budget, with explanations. (4) Close package — the above, formatted for controller review and sign-off."

**Guardrails หลัก**:
- **Supporting documents are untrusted**: reader workers (Read/Grep only, no MCP, no Write) ที่อ่าน invoices/statements
- **No GL posting**: Agent draft JEs; posting require controller approval outside the agent
- **Cite every accrual**: ทุก accrual entry ต้องมี support reference (invoice, contract, estimate)

## Skills ที่ใช้

| Skill | บทบาท |
|-------|------|
| **accrual-schedule** | สร้าง accrual entries (expenses, revenues) กับ support ref และ JE draft |
| **roll-forward** | สร้าง roll-forward schedules (balance sheet items) |
| **variance-commentary** | วิเคราะห์ P&L และ B/S flux, explain major movements |
| **audit-xls** | ตรวจสอบ accruals & roll-forwards balance to GL |
| **xlsx-author** | สร้าง Excel close package |

## Workflow (ขั้นตอนการทำงาน)

1. **Pull trial balance** — GL MCP ดึง trial balance สำหรับ entity และ period
2. **Build accruals** — dispatch workers สำหรับแต่ละ accrual type:
   - **Accrued expenses**: utility bills, rent, legal fees, audit fees (month usage แต่ invoice ยังมาไม่ถึง)
   - **Revenue accruals**: management fees (accrued monthly หรือ quarterly), performance fees (estimated ณ period end)
   - **Deferred revenue reversals**: ยอด ค้างจากเดือนที่แล้วว่าได้รับแล้วหรือยัง
3. **Build roll-forwards** — สำหรับ balance sheet items:
   - Fixed assets: begin + capital additions − depreciation − disposals = end
   - Intangibles: begin + amortization = end
   - Deferred tax: begin + tax provision impact = end
4. **Draft variance commentary** — analyze ทุก line > threshold (เช่น >5% variance จาก prior month หรือ budget):
   - "Revenue up $2M vs. Dec 2023 — driven by (i) management fees +$1.5M (fees on +$500M AUM), (ii) performance fees +$500K"
   - "Operating expense up $500K vs. budget — mainly (i) professional fees +$300K (additional audit for new fund), (ii) technology costs +$200K (system upgrade)"
5. **Assemble package** — poster format close package ให้ controller ready for sign-off

## ตัวอย่างการใช้งาน

**สถานการณ์**: December 2024 close สำหรับ fund management company

**User Input**:
```
Entity: Apex Capital Management, Inc.
Period: Dec 2024 (2024-12)
```

**Agent output**:
1. **Accrual_Schedule_Dec_2024.xlsx** —
   | Description | Amount | Support | JE Draft | GL Account |
   |-------------|--------|---------|----------|-----------|
   | Accrued management fees | $1,200,000 | Fund agreement (1.5% on $80M AUM) | Dr. Receivable, Cr. Fee Revenue | 4100 |
   | Accrued performance fees | $300,000 | Fund performance report (6% of $5M gain) | Dr. Receivable, Cr. Fee Revenue | 4200 |
   | Accrued salaries (Dec payroll) | $500,000 | Payroll summary (monthly staff) | Dr. Compensation, Cr. Payable | 2100 |
   | Accrued audit fees | $50,000 | Audit engagement letter (1 month of $600K annual) | Dr. Professional Fee, Cr. Payable | 5300 |
   | Reverse prior period accruals | ($1,100,000) | Dec 2023 accruals paid Jan 2024 | Dr. Payable, Cr. Receivable | various |

2. **Roll_Forward_Schedules_Dec_2024.xlsx** —
   
   | Account | Nov 30 Balance | Additions | Depreciation/Amortization | Disposals | Dec 31 Balance |
   |---------|-----------------|-----------|------------------------|-----------|-----------------|
   | Furniture & Fixtures | $200,000 | $0 | ($2,000) | $0 | $198,000 |
   | Computer Equipment | $150,000 | $30,000 | ($5,000) | $0 | $175,000 |
   | Leasehold Improvements | $100,000 | $0 | ($1,000) | $0 | $99,000 |
   | Accumulated Depreciation | ($250,000) | — | ($8,000) | — | ($258,000) |
   | **Net Fixed Assets** | **$200,000** | **$30,000** | **($8,000)** | **$0** | **$222,000** |

3. **Variance_Commentary_Dec_2024.txt** —
   
   **P&L Variance vs. December 2023:**
   
   - **Revenue**: $2,000,000 (Dec 2024) vs. $1,500,000 (Dec 2023) = +$500,000 (+33%)
     - Drivers: (i) Management fee +$200K (AUM growth from $75M to $80M), (ii) Performance fees +$300K (two funds exceeded performance targets in Dec)
   
   - **Operating Expenses**: $1,200,000 (Dec 2024) vs. $950,000 (Dec 2023) = +$250,000 (+26%)
     - Drivers: (i) Compensation +$150K (hired 2 new analysts), (ii) Technology & Systems +$100K (cloud migration project)
   
   - **Net Income**: $800,000 (Dec 2024) vs. $550,000 (Dec 2023) = +$250,000 (+45%)
   
   **Balance Sheet Variance vs. December 2023:**
   
   - **Cash**: $1,500,000 (Dec 2024) vs. $2,000,000 (Dec 2023) = −$500,000 (−25%)
     - Drivers: LP distributions paid ($400K), capital investment in infrastructure ($100K)
   
   - **AUM & Fee Receivables**: Up $500K (AUM grew to $80M)

4. **Close_Package_Dec_2024.xlsx** (staged for controller) —
   - Summary of accruals with JE drafts
   - Roll-forward ties to GL
   - Variance analysis and explanations
   - Checklist for controller review:
     - [ ] Accruals reviewed and approved
     - [ ] Roll-forwards balance to GL
     - [ ] Variance explanations reasonable
     - [ ] No outstanding reconciling items
     - [ ] Trial balance balanced after accruals

Agent สำหรับ sign-off: "December close package พร้อม — accruals $1.05M, roll-forwards balanced, net income up 45% YoY driven by fee growth. ต้องการให้ฉันหรือ controller post JEs?"

## Files

**Key files in plugin directory**:
```
plugins/agent-plugins/month-end-closer/
├── .claude-plugin/plugin.json
├── agents/month-end-closer.md (system prompt)
└── skills/
    ├── accrual-schedule/
    ├── roll-forward/
    ├── variance-commentary/
    ├── audit-xls/
    └── xlsx-author/
```
