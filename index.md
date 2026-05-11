---
title: หน้าแรก
layout: home
nav_order: 1
description: "เอกสารภาษาไทยของ Claude for Financial Services — ครอบคลุมทุกโฟลเดอร์ ทุก agent ทุก vertical"
permalink: /
---

# Claude for Financial Services — คู่มือภาษาไทย
{: .fs-9 }

เอกสารแปลและอธิบายแบบครบทุกโฟลเดอร์ของ [`anthropics/financial-services`](https://github.com/anthropics/financial-services) — reference implementation ของ AI agents และ skills สำหรับงานสถาบันการเงิน
{: .fs-6 .fw-300 }

[เริ่มอ่าน — ภาพรวม](./01-overview/){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[ดู Source บน GitHub](https://github.com/anthropics/financial-services){: .btn .fs-5 .mb-4 .mb-md-0 }

---

## คืออะไร?

**Claude for Financial Services** เป็น reference implementation ที่ Anthropic ปล่อยเป็น open source สำหรับงาน **สถาบันการเงิน (Financial Services Industry / FSI)** — ครอบคลุม

- **Investment Banking** (วาณิชธนกิจ) — pitch deck, M&A, IPO
- **Equity Research** (วิเคราะห์หุ้น) — earnings model, initiation report
- **Private Equity** (กองทุนเอกชน) — LBO, portfolio monitoring
- **Wealth Management** (จัดการความมั่งคั่ง) — meeting prep, client review
- **Fund Administration** (ผู้ดูแลกองทุน) — NAV, GL reconciliation
- **Operations** (ปฏิบัติการ) — KYC, AML, audit

## โครงสร้างเอกสารชุดนี้

| หมวด | เนื้อหา |
|------|---------|
| [01 ภาพรวม](./01-overview/) | What/Why/Who — เข้าใจ project ใน 5 นาที |
| [02 โครงสร้างโฟลเดอร์](./02-folders/) | อธิบายทุกโฟลเดอร์ใน repo ต้นทาง |
| [03 Agent Plugins (10 ตัว)](./03-agents/) | 10 named agents — pitch, earnings, KYC, etc. |
| [04 Vertical Plugins (7 ตัว)](./04-verticals/) | 7 หมวดอุตสาหกรรม |
| [05 Partner Plugins](./05-partners/) | LSEG, S&P Global |
| [06 Managed Agent Cookbooks](./06-cookbooks/) | สูตร deploy ผ่าน Managed Agents API |
| [07 Scripts](./07-scripts/) | 6 ไฟล์ Python/Shell ที่ใช้บริหาร repo |
| [08 Microsoft 365 Add-in](./08-msft365/) | plugin สำหรับ Excel/PowerPoint |
| [09 การ Deploy](./09-deployment/) | Cowork vs Claude Code vs Managed Agents API |
| [10 การ Development](./10-development/) | Manifest validation, skill sync, CI |

## หลักการ 5 อย่างที่อยู่เบื้องหลัง

1. **Single source, two runtimes** — เขียน skill ใน vertical ครั้งเดียว → ใช้ได้ทั้ง Cowork plugin และ Managed Agents API
2. **Quality ด้วย manifest validation** — ไม่มี unit test แต่มี `check.py` ที่ลิ้นต์ YAML/JSON และตรวจ cross-reference
3. **Hard-allowlist orchestration** — การส่งงานระหว่าง agent ผ่าน schema validation เข้มงวด ไม่ trust output ของ LLM
4. **Domain validators** — DCF validator เช็คว่า terminal growth < WACC, WACC อยู่ในช่วงที่สมเหตุสมผล
5. **MCP catalog over web search** — 11 ผู้ให้บริการข้อมูลการเงินต่อตรง (Daloopa, FactSet, S&P, Moody's, etc.)

## เอกสารนี้สำหรับใคร

- **นักพัฒนา** ที่อยากเข้าใจวิธี build agent สำหรับงานการเงิน
- **นักวิเคราะห์การเงิน** ที่อยากรู้ว่า AI จะมาช่วยงาน workflow ของตัวเองยังไง
- **ทีม AI ใน FI ไทย** ที่ต้องการ reference สำหรับสร้าง agent ของตัวเอง
- **นักเรียน/นักศึกษา** ที่อยากเห็นตัวอย่าง production-grade Anthropic SDK usage

---

## ข้อสำคัญทางกฎหมาย (Disclaimer)

เอกสารชุดนี้เป็นการ **แปลและอธิบาย** ของ open source repository ต้นทาง ไม่ได้เป็นคำแนะนำทางการเงิน ไม่ได้รับรองโดย Anthropic หรือผู้ให้บริการข้อมูลใดๆ ที่กล่าวถึง

โค้ดต้นทางอยู่ภายใต้ license ตามที่ระบุใน [LICENSE](https://github.com/anthropics/financial-services/blob/main/LICENSE) ของ repo ต้นทาง

---

*แปล/เรียบเรียงโดย [Apex Oracle](https://github.com/wutgii-worawut/apex-oracle) — ผู้เก็บความรู้ทุกยุค 🌳📚*
