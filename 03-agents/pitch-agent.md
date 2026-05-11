---
title: pitch-agent
parent: 03 Agent Plugins
nav_order: 1
---

# Pitch Agent

## ทำอะไร

Pitch Agent เป็น investment banking agent ที่สร้าง pitch deck และ valuation model แบบครบวงจร สำหรับการนำเสนอธุรกิจ (M&A, strategic alternatives) ให้กับ client ให้แล้วเสร็จจากหนึ่งชื่อ ticker และสถานการณ์ยุทธศาสตร์ โดยอัตโนมัติดึงข้อมูล comps และ precedent transactions จากข้อมูลตลาด สร้าง DCF model ใน Excel และสร้าง branded pitch deck พร้อมแผนภูมิที่เชื่อมโยงกับ model

## Use Case

**สถานการณ์จริง**: MD ของธนาคารลงทุนต้องการ first-draft pitch เพื่อนำเสนอกรรมการบริษัท (target company) เกี่ยวกับสถานการณ์ strategic alternatives ภายใน 2-3 ชั่วโมง Pitch Agent จะดึงข้อมูล trading comps และ M&A precedents จากระบบ CapIQ สร้าง Excel workbook พร้อม valuation football field และจัดเรียง PowerPoint deck บนเทมเพลตของธนาคาร

**ผู้ใช้**: MD, Senior Banker, Associate ที่ต้องการ first-draft pitch deck

**สิ่งที่ได้รับ**: Excel valuation workbook (comps, precedents, DCF, LBO, football field) + PowerPoint pitch deck พร้อมแผนภูมิ live-linked

## System Prompt (ส่วนเด่น)

Agent นี้มีบุคลิก "senior investment banking associate" ที่เป็นเจ้าของ end-to-end pitch drafting:

> "Given a target company ticker/name and a one-line situation, you deliver two artifacts: (1) Excel valuation workbook — trading comps, precedent transactions, DCF, and a football-field summary. Every output cell is a live formula traceable to an input. (2) Pitch deck — populated on the bank's PowerPoint template."

**Guardrails หลัก**:
- **Cite every number**: ทุกตัวเลขต้องมาจากฐานข้อมูล CapIQ หรือการไฟลิ่ง ถ้าไม่ได้มาจากแหล่งที่มา ให้ทำเครื่องหมาย `[UNSOURCED]`
- **No external communications**: Agent นี้ไม่มี email หรือ messaging tools
- **Stop and surface for review**: หลังจากสร้าง Excel model เสร็จและหลังจากสร้าง deck เสร็จ ต้องรอ banker อนุมัติ

## Skills ที่ใช้

| Skill | บทบาท |
|-------|------|
| **sector-overview** | เขียน company snapshot และ strategic-rationale narrative |
| **comps-analysis** | สร้าง trading comps และ precedent transactions table พร้อม metric definitions |
| **lbo-model** | สร้าง illustrative LBO model ที่ entry/exit assumptions |
| **dcf-model** | สร้าง DCF model (projection, terminal value, WACC) |
| **3-statement-model** | สร้าง integrated P&L, Balance Sheet, Cash Flow |
| **audit-xls** | ตรวจสอบ Excel model ให้ balance เท่ากัน ไม่มี hardcodes ในตัวเลข |
| **pitch-deck** | นำตัวเลข Excel มาใส่ใน PowerPoint template |
| **ib-check-deck** | ตรวจสอบ QC deck (totals tie, footnotes, dates) |
| **pptx-author** | สร้าง PowerPoint |
| **deck-refresh** | refresh deck เมื่อ model เปลี่ยน |
| **xlsx-author** | สร้าง Excel |

## Workflow (ขั้นตอนการทำงาน)

1. **Scope the ask** — ยืนยันชื่อบริษัท sector และสถานการณ์ ระบุ 5-8 trading comps และ 5-10 precedent transactions ที่เกี่ยวข้องมากที่สุด
2. **Write situation overview** — invoke `sector-overview` เพื่อเขียน company snapshot และ strategic-rationale narrative
3. **Pull data** — ใช้ CapIQ MCP ดึง trading multiples, precedent data, latest filings ของ target
4. **Spread peer set** — invoke `comps-analysis` วาง trading comps และ precedent transactions พร้อม metric definitions
5. **Build sponsor case** — invoke `lbo-model` สำหรับ illustrative LBO ที่ market leverage
6. **Build rest of model** — invoke `dcf-model` และ `3-statement-model` ตามแนว `audit-xls` conventions
7. **Generate football field** — คำนวณ Min/Median/Max จาก comps, precedents, DCF, LBO
8. **Populate deck** — invoke `pitch-deck` นำ Excel data ใส่ PowerPoint template
9. **Run deck QC** — invoke `ib-check-deck` ตรวจสอบ totals tie, footnotes, dates consistent

## ตัวอย่างการใช้งาน

**สถานการณ์**: Banker บอก "สร้าง pitch deck สำหรับ ABC Corp (NYSE: ABC) — exploring strategic alternatives situation"

**User Input**:
```
Ticker: ABC
Situation: exploring strategic alternatives
Target date: Q3 2024
```

**Agent output**:
1. **ABC_Valuation_Model.xlsx** — 
   - Comps sheet: 8 trading comps ที่เลือก (multiples, description)
   - Precedents sheet: 10 M&A comps
   - DCF sheet: projection 5 ปี, terminal value, WACC calculation
   - LBO sheet: sources & uses, debt schedule, returns sensitivity
   - Football field sheet: comps range, precedents range, DCF value, LBO range
   
2. **ABC_Pitch_Deck.pptx** —
   - Cover slide
   - Table of contents
   - Company snapshot (ธุรกิจ, ตลาด, positioning)
   - Strategic rationale (why now, what's changed)
   - Valuation summary + football field chart
   - Detailed comps page
   - Detailed precedents page
   - Process overview

Agent สามารถปรึกษา Banker หลังจากแต่ละ artifact: "Excel model พร้อมแล้ว - ต้องการให้ฉันดำเนินการต่อกับ deck หรือ?"

## Files

**Key files in plugin directory**:
```
plugins/agent-plugins/pitch-agent/
├── .claude-plugin/plugin.json
├── agents/pitch-agent.md (system prompt)
└── skills/
    ├── sector-overview/
    ├── comps-analysis/
    ├── lbo-model/
    ├── dcf-model/
    ├── 3-statement-model/
    ├── audit-xls/
    ├── pitch-deck/
    ├── ib-check-deck/
    ├── pptx-author/
    ├── deck-refresh/
    └── xlsx-author/
```
