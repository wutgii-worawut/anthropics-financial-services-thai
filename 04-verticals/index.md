---
title: 04 Vertical Plugins
nav_order: 5
has_children: true
permalink: /04-verticals/
---

# Vertical Plugins (7 หมวดอุตสาหกรรม)

`plugins/vertical-plugins/` — **source of truth** ของ skill ทั้งหมด แบ่งตามอุตสาหกรรม

| # | Vertical | คำอธิบาย |
|---|----------|----------|
| 1 | [financial-analysis](./financial-analysis.md) | core skills — DCF, comps, LBO (ใช้ร่วมกับ vertical อื่น) |
| 2 | [investment-banking](./investment-banking.md) | วาณิชธนกิจ — pitch, M&A, IPO |
| 3 | [equity-research](./equity-research.md) | วิเคราะห์หุ้น — earnings, initiation |
| 4 | [private-equity](./private-equity.md) | กองทุนเอกชน — LBO, portfolio |
| 5 | [wealth-management](./wealth-management.md) | บริหารความมั่งคั่ง — meeting prep |
| 6 | [fund-admin](./fund-admin.md) | ผู้ดูแลกองทุน — NAV, statement audit |
| 7 | [operations](./operations.md) | ปฏิบัติการ — KYC, GL, AML |

## โครงสร้างมาตรฐาน

```
plugins/vertical-plugins/<vertical>/
├── .claude-plugin/
│   └── plugin.json
├── commands/           # slash commands เช่น /dcf, /comps
├── hooks/              # pre/post hooks (ถ้ามี)
└── skills/             # SOURCE OF TRUTH ของ skill ในหมวดนี้
    └── <skill>/
        └── SKILL.md
```

## ปรัชญา "Vertical-First"

- **Skill** เขียนครั้งเดียวใน vertical → ถูก `sync-agent-skills.py` คัดลอกไปยัง agent plugins ที่ใช้
- การแก้ไขจึงทำที่ vertical เท่านั้น เพื่อหลีกเลี่ยง drift
- ตัวลินต์ (`check.py`) จะแจ้งเตือนถ้าสำเนาใน agent ต่างจาก source ใน vertical
