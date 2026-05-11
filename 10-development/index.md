---
title: 10 การ Development
nav_order: 11
has_children: true
permalink: /10-development/
---

# การ Development

หมวดนี้สำหรับคนที่อยาก **แก้ไข / fork / contribute** เข้า repo

## หัวข้อ

- [Manifest Validation](./manifest-validation.md) — `check.py` ทำอะไร เช็คอะไรบ้าง
- [Skill Sync Workflow](./skill-sync.md) — ทำไมต้อง sync และทำยังไง
- [Output Schema Validation](./schema-validation.md) — validate JSON output ของ subagent
- [Security & CI](./security-ci.md) — Gitleaks secret scan + dry-run deploy
- [การ Fork และ Customize](./fork-and-customize.md) — how to make it yours

## Quality Gates ก่อน Merge

```
push → CI → ▶ Gitleaks secret scan
              ▶ check.py (manifest lint)
              ▶ sync-agent-skills.py --dry-run (drift detection)
              ▶ test-cookbooks.sh (dry-run deploy)
              ▶ validate.py (schema validation)
```

ทุกอย่างต้อง **green** ก่อนจะ merge เข้า main
