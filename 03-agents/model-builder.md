---
title: model-builder
parent: 03 Agent Plugins
nav_order: 4
---

# Model Builder

## ทำอะไร

Model Builder เป็น financial modeling specialist agent ที่สร้าง DCF, LBO, three-statement, และ trading comps models จาก scratch สำหรับบริษัท/ticker ที่กำหนด ใช้เมื่อต้องการ clean institutional-quality model ใหม่ แตกต่างจาก earnings-reviewer ที่ update model เดิม model-builder สร้างจากศูนย์

## Use Case

**สถานการณ์จริง**: Analyst ต้องการสร้าง DCF model สำหรับการประเมิน acquisition target ใหม่ที่ไม่เคยมี model มาก่อน ระบุ ticker, model type (DCF), และ assumption set (terminal growth 2%, WACC 8%) Model Builder ดึง historical data จาก CapIQ/Daloopa, สร้าง Excel พร้อม projection 5 ปี, terminal value calculation, WACC build, sensitivity tables — ทั้งหมดใช้ formula ไม่มี hardcoded numbers ใน calc cells

**ผู้ใช้**: Investment Banker, Equity Research Analyst, Corporate Development Manager

**สิ่งที่ได้รับ**: Fully linked Excel model (DCF / LBO / 3-statement / Comps) พร้อมสีตามแบบแผน blue/black/green

## System Prompt (ส่วนเด่น)

Agent นี้มีบุคลิก "financial modeling specialist":

> "Given a ticker, model type, and assumption set, you deliver a fully linked Excel workbook: (1) DCF — projection period, terminal value, WACC build, sensitivity tables. (2) LBO — sources & uses, debt schedule, returns waterfall, IRR/MOIC sensitivities. (3) Three-statement — integrated IS/BS/CF with working capital and debt schedules. (4) Comps — trading multiples table with summary statistics."

**Guardrails หลัก**:
- **Every output is a formula**: ห้ามมี hardcoded numbers ในตัวเลข calc cells
- **Cite every input**: Assumptions ต้องมี source หรือติดป้าย `[ASSUMPTION]`
- **Stop and surface**: หลังจาก model build และ audit complete ต้องรอ user review ก่อน sensitize

## Skills ที่ใช้

| Skill | บทบาท |
|-------|------|
| **dcf-model** | สร้าง DCF — projection, terminal value, WACC, sensitivity |
| **lbo-model** | สร้าง LBO — sources & uses, debt schedule, returns, IRR/MOIC sensitivity |
| **3-statement-model** | สร้าง integrated P&L, B/S, CF พร้อม WC schedules |
| **comps-analysis** | สร้าง trading comps table กับ summary statistics |
| **audit-xls** | ตรวจสอบ balance checks, no circular refs (นอกจากที่ intentional), outputs trace to inputs |
| **xlsx-author** | สร้าง Excel structure |

## Workflow (ขั้นตอนการทำงาน)

1. **Pull inputs** — ใช้ CapIQ/Daloopa MCP ดึง historical P&L, B/S, CF, consensus estimates, latest filings
2. **Build the model** — invoke สำหรับ model type ที่ request:
   - **DCF**: 5-year projection, terminal value (growth method or exit multiple), WACC calculation
   - **LBO**: sources & uses, debt schedule, returns waterfall, entry/exit assumptions
   - **Three-statement**: integrated with working capital (receivables, payables, inventory)
   - **Comps**: 10-15 peer companies, NTM/LTM multiples
3. **Audit** — invoke `audit-xls` ตรวจสอบ balance sheet tie (assets = liabilities + equity), CF reconciliation, no broken formula
4. **Sensitize** — build standard sensitivity tables (WACC ±1%, terminal growth ±0.5%, etc.)
5. **Surface for review** — stop หลังจาก build & audit, ให้ user approve ก่อน sensitize

## ตัวอย่างการใช้งาน

**สถานการณ์**: Analyst จะ value XYZ Corp (ม.ค. 2025) ไม่มี existing model

**User Input**:
```
Ticker: XYZ
Model type: DCF
Assumptions: terminal growth 2.5%, WACC 8.5%, projection 5 years
```

**Agent output**:
1. **XYZ_DCF_Model.xlsx** —
   - **Inputs sheet**: Historical financials (2022-2024), assumptions (growth, WACC), sources
   - **Projection sheet**: P&L + B/S + CF (2025-2029) — ทุกตัวเลขใช้ formula
   - **WACC sheet**: cost of equity (CAPM), cost of debt, blended WACC calculation ที่ detail
   - **DCF sheet**: 
     - Unlevered FCF projection
     - Terminal value (2.5% perpetual growth method)
     - PV ของ FCF + terminal value
     - Enterprise value, less net debt, = Equity value
   - **Summary sheet**: DCF implied price per share
   - **Sensitivity tables**: WACC ±1%, terminal growth ±0.5%, revenue growth ±2%
   - **Color scheme**: Blue (inputs), Black (calculations), Green (outputs)

Agent ส่งข้อความ: "DCF model พร้อมแล้ว - implied value $45/share ที่ WACC 8.5% — ต้องการให้ฉันทำ LBO model หรือ comps ด้วยหรือไม่?"

## Files

**Key files in plugin directory**:
```
plugins/agent-plugins/model-builder/
├── .claude-plugin/plugin.json
├── agents/model-builder.md (system prompt)
└── skills/
    ├── dcf-model/
    ├── lbo-model/
    ├── 3-statement-model/
    ├── comps-analysis/
    ├── audit-xls/
    └── xlsx-author/
```
