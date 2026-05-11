---
title: Skill Sync Workflow
parent: 10 การ Development
nav_order: 2
---

# Skill Sync — sync-agent-skills.py

## ทำไมต้อง Sync

Skills ถูก author ครั้งเดียว ใน **vertical-plugins** source:

```
plugins/vertical-plugins/financial-analysis/skills/comps-analysis/
  ↑ SINGLE SOURCE OF TRUTH
```

But bundled ใน agent plugins:

```
plugins/agent-plugins/pitch-agent/skills/comps-analysis/          (copy)
plugins/agent-plugins/gl-reconciler/skills/comps-analysis/        (copy)
plugins/agent-plugins/model-builder/skills/comps-analysis/        (copy)
```

ถ้า edit skill ใน vertical → ต้อง propagate ไปยัง agents ทั้งหมด

ไม่งั้น agent A มี skill v1, agent B มี skill v2 — inconsistent behavior

## sync-agent-skills.py Algorithm

Script ทำ:

1. Index ทุก skill ใน `plugins/vertical-plugins/*/skills/<name>/`
2. หา bundled copies ใน `plugins/agent-plugins/*/skills/<name>/`
3. `shutil.rmtree()` old copy
4. `shutil.copytree()` new copy จาก source

```python
src_by_name: dict[str, Path] = {}
for sk in VERTICALS.glob("*/skills/*"):
  if sk.is_dir():
    src_by_name[sk.name] = sk

for bundled in sorted(AGENTS.glob("*/skills/*")):
  if not bundled.is_dir():
    continue
  src = src_by_name.get(bundled.name)
  if not src:
    error(f"no vertical source for {bundled.name}")
    continue
  shutil.rmtree(bundled)
  shutil.copytree(src, bundled)
  synced += 1
```

## ใช้งาน

เมื่อคุณ edit skill ใน vertical:

```bash
# 1. Edit source
vim plugins/vertical-plugins/financial-analysis/skills/dcf-model/SKILL.md

# 2. Run sync
python3 scripts/sync-agent-skills.py
```

Output:

```
synced 8 bundled skill dir(s) from vertical-plugins/
```

Command นี้:

```
plugins/vertical-plugins/financial-analysis/skills/dcf-model/
  → plugins/agent-plugins/pitch-agent/skills/dcf-model/
  → plugins/agent-plugins/model-builder/skills/dcf-model/
  → plugins/agent-plugins/earnings-reviewer/skills/dcf-model/
  ... (any agent that bundles dcf-model)
```

## Git Workflow

```bash
# Edit skill ใน vertical
vim plugins/vertical-plugins/financial-analysis/skills/dcf-model/SKILL.md
vim plugins/vertical-plugins/financial-analysis/skills/dcf-model/examples.yaml

# Run sync
python3 scripts/sync-agent-skills.py

# Stage + commit
git add plugins/
git commit -m "feat(dcf-model): add IRR sensitivity analysis

- Add sensitivity table for discount rate ±1%
- Show Monte Carlo simulation option
- Updated in 8 agent bundles via sync script"
```

## Missing Vertical Source

ถ้า bundled skill ไม่มี vertical source — error + exit 1:

```bash
python3 scripts/sync-agent-skills.py
```

Output:

```
synced 8 bundled skill dir(s) from vertical-plugins/
WARN: no vertical source found for:
  - plugins/agent-plugins/pitch-agent/skills/custom-deck-formatter
```

Fix: แม่า skill ไป vertical แล้ว sync อีกที:

```bash
# Copy bundled to vertical (reverse operation)
cp -r plugins/agent-plugins/pitch-agent/skills/custom-deck-formatter \
      plugins/vertical-plugins/investment-banking/skills/

# Or delete if it's only in agent-plugins by mistake
rm -r plugins/agent-plugins/pitch-agent/skills/custom-deck-formatter

# Sync again
python3 scripts/sync-agent-skills.py
```

## Before Merging PR

ตามขั้นตอน:

1. **Edit skill ใน vertical**
2. **Run sync**
3. **Run check**
4. **Commit**

```bash
# 1. Edit
vim plugins/vertical-plugins/private-equity/skills/ic-memo/SKILL.md

# 2. Sync
python3 scripts/sync-agent-skills.py

# 3. Check
python3 scripts/check.py

# 4. Commit
git add plugins/
git commit -m "update ic-memo skill: add ESG assessment"

# 5. Push
git push origin feature-branch
```

ถ้า check fail → fix ก่อน push

## Danger: Manual Edits ❌

**ไม่ให้ edit bundled skill directly:**

```bash
# ❌ DON'T
vim plugins/agent-plugins/pitch-agent/skills/comps-analysis/SKILL.md
```

Why: next sync จะ overwrite

**ทำให้ edit source ใน vertical:**

```bash
# ✅ DO
vim plugins/vertical-plugins/financial-analysis/skills/comps-analysis/SKILL.md
python3 scripts/sync-agent-skills.py
```

## Skill Lifecycle

```
1. Create skill ใน vertical
   └─ plugins/vertical-plugins/<vert>/skills/<name>/

2. Bundle in agents ต่อรู้ใจ
   └─ sync-agent-skills.py copy ไปยัง
      plugins/agent-plugins/<agent>/skills/<name>/

3. Edit? Edit ใน vertical เท่านั้น
   └─ re-run sync

4. Delete? Remove จาก vertical
   └─ sync จะ delete bundled copies
   └─ agents ที่ใช้มันต้อง remove reference จาก agent.yaml
```

---

**Next:** [Schema validation — validate subagent output](schema-validation.md)
