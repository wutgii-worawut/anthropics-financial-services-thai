---
title: sync-agent-skills.py
parent: 07 Scripts
nav_order: 3
---

# sync-agent-skills.py — Sync Bundled Skills จาก Vertical Sources

## หน้าที่ (Purpose)

Skill ที่ใช้โดย agents จะเก็บไว้ 2 ที่:
1. **Source**: `plugins/vertical-plugins/*/skills/<name>/` — ที่เก็บ master copy (เป็น source of truth)
2. **Bundled**: `plugins/agent-plugins/<slug>/skills/<name>/` — vendored copy สำหรับแต่ละ agent

Script นี้ sync bundled copies ไปทันสมัยกับ vertical sources หลังจากแก้ไข skill ใดใน vertical

**เมื่อควรใช้:**
- หลังจากแก้ไข skill ใดใน `vertical-plugins/`
- หลังจาก `scripts/check.py` รายงานว่า bundled skill drift ไปจาก source
- ก่อน commit เพื่อให้ agent bundles ตรงกับ vertical source

## Usage

```bash
python3 scripts/sync-agent-skills.py
```

ไม่มี arguments — script อ่าน directory structure อัตโนมัติ

**ข้อกำหนด:**
- Python 3.7+
- No extra dependencies (uses standard `shutil`)

**รันจาก:** Repository root

## ภายในทำอะไร (How It Works)

### 1. Index Vertical Skills (บรรทัด 20-24)
สร้าง dictionary ของทุก skill ใน vertical-plugins ที่ key คือชื่อ skill:

```python
src_by_name: dict[str, Path] = {}
for sk in VERTICALS.glob("*/skills/*"):
    if sk.is_dir():
        src_by_name[sk.name] = sk
```

ตัวอย่าง:
```
src_by_name = {
  "fundamental-analysis": Path("plugins/vertical-plugins/equities/skills/fundamental-analysis"),
  "ratio-calc": Path("plugins/vertical-plugins/fixed-income/skills/ratio-calc"),
  ...
}
```

### 2. Iterate Agent Bundles (บรรทัด 28-37)
วนลูปแต่ละ bundled skill ภายใต้ `plugins/agent-plugins/<agent>/skills/<name>/`:

```python
for bundled in sorted(AGENTS.glob("*/skills/*")):
    if not bundled.is_dir():
        continue
    src = src_by_name.get(bundled.name)
    if not src:
        missing.append(str(bundled.relative_to(ROOT)))
        continue
    shutil.rmtree(bundled)      # Delete old bundled copy
    shutil.copytree(src, bundled)  # Copy fresh from source
    synced += 1
```

สำหรับแต่ละ bundled skill:
- ถ้า source มีอยู่ → delete bundled copy เก่า → copy fresh
- ถ้า source ไม่มีอยู่ → บันทึกเป็น missing

### 3. Report (บรรทัด 39-44)
พิมพ์สรุป:

```python
print(f"synced {synced} bundled skill dir(s) from vertical-plugins/")
if missing:
    print("WARN: no vertical source found for:", file=sys.stderr)
    for m in missing:
        print(f"  - {m}", file=sys.stderr)
    sys.exit(1)
```

## Output

```
synced 12 bundled skill dir(s) from vertical-plugins/
```

หากพบ skill ที่ไม่มี source:
```
synced 11 bundled skill dir(s) from vertical-plugins/
WARN: no vertical source found for:
  - plugins/agent-plugins/pitch-agent/skills/orphaned-skill
  - plugins/agent-plugins/gl-reconciler/skills/deprecated-tool
```

**Exit codes:**
- `0` — sync สำเร็จ (หรือมี missing แต่ complete warnings ออกมา)
- `1` — มี missing skills (ต้องลบ bundled copies ที่ orphaned หรือเพิ่ม source)

## ตัวอย่าง (Example)

```bash
# You edit a vertical skill
$ vim plugins/vertical-plugins/equities/skills/fundamental-analysis/analyzer.py

# Check validation
$ python3 scripts/check.py
FAIL — 1 issue(s) across 120 file(s):

  ✗ bundled-skill: plugins/agent-plugins/pitch-agent/skills/fundamental-analysis: 
    drifted from plugins/vertical-plugins/equities/skills/fundamental-analysis 
    (run scripts/sync-agent-skills.py)

# Sync bundled copies
$ python3 scripts/sync-agent-skills.py
synced 12 bundled skill dir(s) from vertical-plugins/

# Validation passes now
$ python3 scripts/check.py
OK — 120 file(s) checked, 0 issues.
```

### Scenario: Removing Deprecated Skill

```bash
# You've removed a skill from vertical
$ rm -rf plugins/vertical-plugins/equities/skills/old-rsi-indicator

# Sync finds it's orphaned
$ python3 scripts/sync-agent-skills.py
synced 11 bundled skill dir(s) from vertical-plugins/
WARN: no vertical source found for:
  - plugins/agent-plugins/pitch-agent/skills/old-rsi-indicator
  - plugins/agent-plugins/valuation-reviewer/skills/old-rsi-indicator

# Option 1: Remove bundled copies manually
$ rm -rf plugins/agent-plugins/*/skills/old-rsi-indicator

# Option 2: Re-add the source if it's still needed
$ mkdir -p plugins/vertical-plugins/equities/skills/old-rsi-indicator
$ cp old-rsi-indicator-backup/* plugins/vertical-plugins/equities/skills/old-rsi-indicator/

# Then sync again
$ python3 scripts/sync-agent-skills.py
synced 12 bundled skill dir(s) from vertical-plugins/
```

## เกี่ยวข้องกับอะไรอีกบ้าง

- **check.py**: บอก error ว่า skill drift — ให้ run sync-agent-skills.py เพื่อแก้ไข
- **Skill development workflow**: 
  1. แก้ไขใน `vertical-plugins/*/skills/<name>/`
  2. Test the skill
  3. Run `sync-agent-skills.py` เพื่อ propagate ไปยังทั้ง agents
  4. Run `check.py` เพื่อตรวจสอบ
  5. Commit both vertical + bundled changes
- **CI/CD**: อาจ run `sync-agent-skills.py` อัตโนมัติหากมี changes ใน vertical-plugins
