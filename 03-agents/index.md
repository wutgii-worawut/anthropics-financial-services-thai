---
title: 03 Agent Plugins
nav_order: 4
has_children: true
permalink: /03-agents/
---

# Agent Plugins (10 ตัว)

`plugins/agent-plugins/` — **10 named agents** ที่เป็น **end-to-end workflow agent** สำหรับงาน FSI

| # | Agent | งานหลัก | Vertical หลัก |
|---|-------|---------|---------------|
| 1 | [pitch-agent](./pitch-agent.md) | สร้าง pitch deck สำหรับ M&A | Investment Banking |
| 2 | [earnings-reviewer](./earnings-reviewer.md) | รีวิว earnings + อัพเดท model | Equity Research |
| 3 | [valuation-reviewer](./valuation-reviewer.md) | ตรวจ DCF / Comps | Equity Research, IB |
| 4 | [model-builder](./model-builder.md) | สร้าง financial model ใน Excel | IB, ER |
| 5 | [market-researcher](./market-researcher.md) | วิจัยตลาด + อุตสาหกรรม | All |
| 6 | [meeting-prep-agent](./meeting-prep-agent.md) | เตรียมข้อมูลก่อนประชุมลูกค้า | Wealth Management |
| 7 | [kyc-screener](./kyc-screener.md) | KYC / AML screening | Operations |
| 8 | [statement-auditor](./statement-auditor.md) | ตรวจงบการเงิน | Fund Admin |
| 9 | [gl-reconciler](./gl-reconciler.md) | กระทบยอด General Ledger | Fund Admin |
| 10 | [month-end-closer](./month-end-closer.md) | ปิดบัญชีสิ้นเดือน | Operations, Fund Admin |

## โครงสร้างมาตรฐานของ agent plugin หนึ่งตัว

```
plugins/agent-plugins/<agent-name>/
├── .claude-plugin/
│   └── plugin.json          # metadata (name, version, description)
├── agents/
│   └── <agent-name>.md      # system prompt + frontmatter
└── skills/                  # synced copy ของ skill จาก vertical
    └── <skill-name>/
        └── SKILL.md
```

**สำคัญ**: skills ใน folder นี้คือ **สำเนา** ที่ sync มาจาก `plugins/vertical-plugins/` ผ่าน script `sync-agent-skills.py` — source of truth อยู่ที่ vertical
