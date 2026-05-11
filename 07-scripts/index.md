---
title: 07 Scripts
nav_order: 8
has_children: true
permalink: /07-scripts/
---

# Scripts

`scripts/` — utility script 6 ตัวที่ใช้บริหาร repo

| Script | ภาษา | หน้าที่ |
|--------|------|---------|
| [check.py](./check.md) | Python | ลินต์ manifest, ตรวจ cross-reference, ตรวจ skill drift |
| [orchestrate.py](./orchestrate.md) | Python | engine สำหรับรัน Managed Agent + จัดการ handoff |
| [sync-agent-skills.py](./sync-agent-skills.md) | Python | คัดลอก skill จาก vertical ไปยัง agent plugin |
| [validate.py](./validate.md) | Python | ตรวจ output ของ subagent ตาม JSON Schema |
| [deploy-managed-agent.sh](./deploy-managed-agent.md) | Shell | deploy cookbook ขึ้น Managed Agents API |
| [test-cookbooks.sh](./test-cookbooks.md) | Shell | dry-run ตรวจ cookbook ทั้งหมดก่อน deploy |

## ทำไมไม่มี unit test?

repo นี้เป็น **reference template** ไม่ใช่ library — ดังนั้นไม่มี pytest/jest แต่ quality ถูก enforce ผ่าน 4 ชั้น:

1. **Manifest lint** (`check.py`) — YAML/JSON syntax + cross-ref
2. **Skill drift** (`check.py`) — สำเนาใน agent ต้องตรงกับ source ใน vertical
3. **Schema validate** (`validate.py`) — output ของ subagent ตาม JSON schema
4. **Dry-run deploy** (`test-cookbooks.sh`) — simulate ทุก cookbook ก่อน deploy จริง

อ่านรายละเอียดที่ [10 Development](../10-development/)
