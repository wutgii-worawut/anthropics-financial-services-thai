---
title: check.py
parent: 07 Scripts
nav_order: 1
---

# check.py — ตรวจสอบรูปร่างของ Manifest ทั้งหมด

## หน้าที่ (Purpose)

Script นี้เป็น linter สำหรับระบบ Managed Agent ทั้งหมด มันตรวจสอบความสมบูรณ์และความถูกต้องของ manifest files (YAML, JSON) และ reference ต่าง ๆ ในโครงสร้าง plugin เพื่อป้องกันการ deploy agent ที่มีข้อมูลหายหรือพัง

**เมื่อควรใช้:**
- ก่อน commit ในการเปลี่ยนแปลง manifest ใด ๆ
- ในขั้นตอน CI/CD เพื่อให้มั่นใจว่า manifest สมบูรณ์ก่อน deploy
- เมื่อแก้ไข skills, system prompts, หรือ callable agents ที่มีการอ้างอิง path

## Usage

```bash
python3 scripts/check.py
```

ไม่มี arguments หรือ flags — script อ่านจาก repository root (`ROOT = Path(__file__).resolve().parents[1]`)

**ข้อกำหนด:**
- Python 3.7+
- `pyyaml` library (`pip install pyyaml`)

**รันจาก:** Repository root

## ภายในทำอะไร (How it Works)

Script ตรวจสอบ 5 ขั้นตอนหลัก (บรรทัด 40-159):

### 1. YAML Parse (บรรทัด 41-47)
วนลูปทุกไฟล์ `*.yaml` ใน `managed-agent-cookbooks/` และพยายาม parse ด้วย `yaml.safe_load()` หากเป็น invalid YAML จะบันทึก error

### 2. JSON Parse (บรรทัด 49-61)
ตรวจสอบ JSON files 3 ประเภท:
- `.claude-plugin/marketplace.json`
- `plugins/**/.claude-plugin/plugin.json`
- `managed-agent-cookbooks/*/steering-examples.json`

### 3. Agent Markdown Frontmatter (บรรทัด 63-77)
ทุกไฟล์ `agent-plugins/*/agents/*.md` ต้องมี YAML frontmatter ที่มี:
- `name` (required)
- `description` (required)

```yaml
---
name: "my-agent"
description: "does something useful"
---
```

### 4. Reference Resolution (บรรทัด 80-143)
ตรวจสอบว่า path ที่อ้างอิงมีอยู่จริง:

- **`system.file`** (บรรทัด 89-92): ไฟล์ system prompt ต้องมีอยู่
- **`skills[].path`** (บรรทัด 94-98): directory ของแต่ละ skill ต้องมีอยู่
- **`skills[].from_plugin`** (บรรทัด 99-102): ต้องมี `skills/` subdirectory
- **`callable_agents[].manifest`** (บรรทัด 104-108): manifest file ของ subagent ต้องมีอยู่

### 4b. Bundled Skills Sync Check (บรรทัด 114-131)
หลังจากแก้ไข skill ที่เก็บใน `vertical-plugins/*/skills/*` จะต้อง sync ไปยังทุก agent-plugin ที่ใช้มัน script นี้ตรวจสอบ:

```python
cmp = filecmp.dircmp(src, bundled)
if cmp.diff_files or cmp.left_only or cmp.right_only:
    err(f"bundled-skill: ... (run scripts/sync-agent-skills.py)")
```

### 4b2. Agent Prose Skill References (บรรทัด 133-143)
ไฟล์ markdown ของ agent ตัวมันเองจะอ้างอิงถึง skill ชื่อต่าง ๆ (เช่น `` `my-skill` ``) script ตรวจสอบว่า skill นั้นมีในชุด bundled skills ของ agent:

```python
for ref in set(re.findall(r"`([a-z0-9]+(?:-[a-z0-9]+)+)`", md.read_text())):
    if ref in src_by_name and ref not in bundle:
        err(f"agent-prose: {rel(md)}: references `{ref}` but ... is not bundled")
```

### 4c. Marketplace Source Paths (บรรทัด 145-150)
แต่ละ plugin ในไฟล์ marketplace.json ต้องชี้ไปยัง directory ที่มี `.claude-plugin/plugin.json`

### 5. Required Files per Managed Agent (บรรทัด 152-158)
ทุก managed-agent directory ต้องมี:
- `agent.yaml`
- `README.md`
- `steering-examples.json`

## Output

```
OK — 127 file(s) checked, 0 issues.
```

หากพบข้อผิดพลาด:
```
FAIL — 3 issue(s) across 120 file(s):

  ✗ YAML parse: managed-agent-cookbooks/pitch-agent/agent.yaml: mapping values are not allowed here
  ✗ ref: managed-agent-cookbooks/gl-reconciler/agent.yaml: system.file -> system.txt (not found)
  ✗ missing: managed-agent-cookbooks/kyc-screener/steering-examples.json
```

**Exit codes:**
- `0` — ไม่พบ error
- `1` — พบ error (แสดงรายการหมด)
- `2` — Missing pyyaml library

## ตัวอย่าง (Example)

```bash
$ python3 scripts/check.py
OK — 127 file(s) checked, 0 issues.

$ echo "invalid: yaml: {" >> managed-agent-cookbooks/test/agent.yaml
$ python3 scripts/check.py
FAIL — 1 issue(s) across 127 file(s):

  ✗ YAML parse: managed-agent-cookbooks/test/agent.yaml: mapping values are not allowed here (line 5)
```

## เกี่ยวข้องกับอะไรอีกบ้าง

- **sync-agent-skills.py**: ถ้าพบ "bundled-skill" error ให้รัน script นี้เพื่อ sync
- **deploy-managed-agent.sh**: ควร run `check.py` ก่อนการ deploy เพื่อให้ manifest พร้อม
- **CI/CD workflow**: โดยปกติเรียก `check.py` ในลำดับแรกของ validation pipeline
