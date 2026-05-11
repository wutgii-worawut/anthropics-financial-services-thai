---
title: check.py
parent: 07 Scripts
nav_order: 1
---

# check.py — ตรวจสอบรูปร่างของ Manifest ทั้งหมด

## หน้าที่ (Purpose)

Script นี้เป็น **linter** สำหรับระบบ Managed Agent ทั้งหมด มันตรวจสอบความสมบูรณ์และความถูกต้องของ manifest files (YAML, JSON) และ reference ต่าง ๆ ในโครงสร้าง plugin เพื่อป้องกันการ deploy agent ที่มีข้อมูลหายหรือพัง

**เมื่อควรใช้:**
- ก่อน `git commit` ในการเปลี่ยนแปลง manifest ใด ๆ
- ในขั้นตอน CI/CD เพื่อให้มั่นใจว่า manifest สมบูรณ์ก่อน deploy
- เมื่อแก้ไข skills, system prompts, หรือ callable agents ที่มีการอ้างอิง path
- หลังจาก rename file หรือ directory ที่อาจแตกการอ้างอิง

## Usage

```bash
python3 scripts/check.py
```

**ไม่มี arguments หรือ flags** — script อ่านจาก repository root (`ROOT = Path(__file__).resolve().parents[1]`)

**ข้อกำหนด:**
- Python 3.7+
- `pyyaml` library (`pip install pyyaml`)

**รันจาก:** Repository root

```bash
cd /path/to/financial-services
python3 scripts/check.py
```

## ขั้นตอนการตรวจสอบ (5 Main Validation Steps)

Script ทำการตรวจสอบ 5 ขั้นตอนหลัก (บรรทัด 40-159):

### Step 1: YAML Parse Validation (บรรทัด 41-47)

```python
# Loop through all *.yaml files in managed-agent-cookbooks/
for yml in sorted(MANAGED.rglob("*.yaml")):
    checked += 1
    try:
        with open(yml) as f:
            yaml.safe_load(f)  # Parse with yaml.safe_load()
    except yaml.YAMLError as e:
        err(f"YAML parse: {rel(yml)}: {e}")
```

**ทำอะไร**:
- วนลูปทุกไฟล์ `*.yaml` ใน `managed-agent-cookbooks/` directory
- พยายาม parse ด้วย `yaml.safe_load()`
- หากเป็น invalid YAML จะบันทึก error

**ตัวอย่าง error**:
```
✗ YAML parse: managed-agent-cookbooks/pitch-agent/agent.yaml: mapping values are not allowed here (line 12)
```

### Step 2: JSON Parse Validation (บรรทัด 49-61)

```python
json_globs = [
    ".claude-plugin/marketplace.json",
    "plugins/**/.claude-plugin/plugin.json",
    "managed-agent-cookbooks/*/steering-examples.json",
]
for pat in json_globs:
    for jf in sorted(ROOT.glob(pat)):
        checked += 1
        try:
            json.loads(jf.read_text())
        except json.JSONDecodeError as e:
            err(f"JSON parse: {rel(jf)}: {e}")
```

**ทำอะไร**:
- ตรวจสอบ JSON files 3 ประเภท:
  1. `.claude-plugin/marketplace.json` — plugin registry
  2. `plugins/**/.claude-plugin/plugin.json` — plugin metadata
  3. `managed-agent-cookbooks/*/steering-examples.json` — agent steering events
- Parse ด้วย `json.loads()`, flag malformed JSON

**ตัวอย่าง error**:
```
✗ JSON parse: managed-agent-cookbooks/earnings-reviewer/steering-examples.json: Expecting ',' delimiter: line 5 column 10
```

### Step 3: Agent Markdown Frontmatter Validation (บรรทัด 63-77)

```python
for md in sorted(PLUGINS.glob("agent-plugins/*/agents/*.md")):
    checked += 1
    text = md.read_text()
    if not text.startswith("---"):
        err(f"frontmatter: {rel(md)}: missing leading ---")
        continue
    try:
        _, fm, _ = text.split("---", 2)  # Extract YAML block
        meta = yaml.safe_load(fm)
        for k in ("name", "description"):
            if k not in meta:
                err(f"frontmatter: {rel(md)}: missing '{k}'")
    except (ValueError, yaml.YAMLError) as e:
        err(f"frontmatter: {rel(md)}: {e}")
```

**ทำอะไร**:
- ทุกไฟล์ `agent-plugins/*/agents/*.md` ต้องมี YAML frontmatter
- Frontmatter ต้องมี required fields: `name`, `description`
- Format:
  ```yaml
  ---
  name: "my-agent"
  description: "does something useful"
  ---
  ```

**ตัวอย่าง error**:
```
✗ frontmatter: plugins/agent-plugins/pitch-agent/agents/pitch-agent.md: missing 'description'
```

### Step 4: Reference Resolution (บรรทัด 80-150)

นี่คือขั้นตอนหลัก — ตรวจสอบว่า paths ที่อ้างอิงมีอยู่จริง

#### Step 4a: Basic Path Resolution (บรรทัด 89-108)

```python
def check_refs(yml: Path) -> None:
    data = yaml.safe_load(yml.read_text()) or {}
    base = yml.parent
    
    # Check system.file
    sys_spec = data.get("system")
    if isinstance(sys_spec, dict) and "file" in sys_spec:
        p = (base / sys_spec["file"]).resolve()
        if not p.is_file():
            err(f"ref: {rel(yml)}: system.file -> {sys_spec['file']} (not found)")
    
    # Check skills[].path
    for s in data.get("skills") or []:
        if isinstance(s, dict) and "path" in s:
            p = (base / s["path"]).resolve()
            if not p.exists():
                err(f"ref: {rel(yml)}: skills.path -> {s['path']} (not found)")
    
    # Check skills[].from_plugin
    if isinstance(s, dict) and "from_plugin" in s:
        p = (base / s["from_plugin"]).resolve()
        if not (p / "skills").is_dir():
            err(f"ref: {rel(yml)}: skills.from_plugin -> {s['from_plugin']} (no skills/ dir)")
    
    # Check callable_agents[].manifest
    for c in data.get("callable_agents") or []:
        if isinstance(c, dict) and "manifest" in c:
            p = (base / c["manifest"]).resolve()
            if not p.is_file():
                err(f"ref: {rel(yml)}: callable_agents.manifest -> {c['manifest']} (not found)")
```

**ตรวจสอบต่อไปนี้** ในทุกไฟล์ agent.yaml และ subagent .yaml:

| Reference Type | Check | Example |
|---|---|---|
| `system.file` | File ต้องมีอยู่ | `system.file: ../../plugins/agent-plugins/pitch-agent/agents/pitch-agent.md` |
| `skills[].path` | Directory/file ต้องมีอยู่ | `skills: [{ path: ./skills/my-skill }]` |
| `skills[].from_plugin` | `skills/` subdirectory ต้องมีอยู่ | `skills: [{ from_plugin: ../../plugins/agent-plugins/pitch-agent }]` |
| `callable_agents[].manifest` | Manifest file ต้องมีอยู่ | `callable_agents: [{ manifest: ./subagents/researcher.yaml }]` |

**ตัวอย่าง error**:
```
✗ ref: managed-agent-cookbooks/pitch-agent/agent.yaml: system.file -> ../../plugins/agent-plugins/pitch-agent/agents/pitch-agent.md (not found)
✗ ref: managed-agent-cookbooks/earnings-reviewer/agent.yaml: callable_agents.manifest -> ./subagents/transcript-reader.yaml (not found)
```

#### Step 4b: Bundled Skills Sync Check (บรรทัด 114-131)

```python
src_by_name = {p.name: p for p in PLUGINS.glob("vertical-plugins/*/skills/*") if p.is_dir()}
for bundled in sorted(PLUGINS.glob("agent-plugins/*/skills/*")):
    if not bundled.is_dir():
        continue
    src = src_by_name.get(bundled.name)
    if not src:
        err(f"bundled-skill: {rel(bundled)}: no vertical-plugins source named '{bundled.name}'")
        continue
    cmp = filecmp.dircmp(src, bundled)
    if cmp.diff_files or cmp.left_only or cmp.right_only:
        err(
            f"bundled-skill: {rel(bundled)}: drifted from {rel(src)} "
            f"(run scripts/sync-agent-skills.py)"
        )
```

**ทำอะไร**:
- หลังจากแก้ไข skill ที่เก็บใน `vertical-plugins/*/skills/*` ต้อง sync ไปยังทุก agent-plugin ที่ใช้มัน
- Script นี้ตรวจสอบ:
  1. ทุก bundled skill ใน `agent-plugins/*/skills/` ต้องมี source ใน `vertical-plugins/*/skills/`
  2. Bundled skill ต้องตรงกับ source (ไฟล์เดียวกัน, ไม่มีไฟล์เพิ่มเติม)

**ตัวอย่าง error**:
```
✗ bundled-skill: plugins/agent-plugins/pitch-agent/skills/tearsheet: drifted from plugins/vertical-plugins/tearsheet (run scripts/sync-agent-skills.py)
```

**วิธีแก**:
```bash
python3 scripts/sync-agent-skills.py
```

#### Step 4b2: Agent Prose Skill References (บรรทัด 133-143)

```python
for md in sorted(PLUGINS.glob("agent-plugins/*/agents/*.md")):
    slug = md.parents[1].name
    sk_dir = PLUGINS / "agent-plugins" / slug / "skills"
    bundle = {p.name for p in sk_dir.iterdir() if p.is_dir()} if sk_dir.is_dir() else set()
    for ref in set(re.findall(r"`([a-z0-9]+(?:-[a-z0-9]+)+)`", md.read_text())):
        if ref in src_by_name and ref not in bundle:
            err(
                f"agent-prose: {rel(md)}: references `{ref}` but "
                f"plugins/agent-plugins/{slug}/skills/{ref}/ is not bundled"
            )
```

**ทำอะไร**:
- Agent markdown files อ้างอิงถึง skills ด้วย backtick notation: `` `skill-name` ``
- Script ตรวจสอบ: ทุก skill ที่อ้างอิงใน markdown ต้องมีอยู่ใน bundled skills ของ agent นั้น
- ใช้ regex `r"`([a-z0-9]+(?:-[a-z0-9]+)+)`"` เพื่อหา skill references

**ตัวอย่าง**: ไฟล์ markdown พูด "ใช้ `comparable-company-analysis` skill"
- Script ตรวจ: `plugins/agent-plugins/pitch-agent/skills/comparable-company-analysis/` ต้องมีอยู่

**ตัวอย่าง error**:
```
✗ agent-prose: plugins/agent-plugins/pitch-agent/agents/pitch-agent.md: references `tearsheet` but plugins/agent-plugins/pitch-agent/skills/tearsheet/ is not bundled
```

#### Step 4c: Marketplace Source Paths (บรรทัด 145-150)

```python
mp = ROOT / ".claude-plugin" / "marketplace.json"
for p in json.loads(mp.read_text()).get("plugins", []):
    src = (ROOT / p["source"]).resolve()
    if not (src / ".claude-plugin" / "plugin.json").is_file():
        err(f"marketplace: {p['name']} source -> {p['source']} (no plugin.json)")
```

**ทำอะไร**:
- `.claude-plugin/marketplace.json` ระบุรายชื่อ plugins ทั้งหมด
- แต่ละ plugin entry มี `"source"` field ที่ชี้ไปยัง directory
- Directory นั้นต้องมี `.claude-plugin/plugin.json` ไฟล์

**ตัวอย่าง error**:
```
✗ marketplace: S&P Global source -> plugins/partner-built/spglobal (no plugin.json)
```

### Step 5: Required Files per Managed Agent (บรรทัด 152-158)

```python
for d in sorted(MANAGED.iterdir()):
    if not d.is_dir():
        continue
    for req in ("agent.yaml", "README.md", "steering-examples.json"):
        if not (d / req).is_file():
            err(f"missing: {rel(d)}/{req}")
```

**ทำอะไร**:
- ทุก managed-agent directory ต้องมี 3 ไฟล์บังคับ:
  1. `agent.yaml` — agent manifest
  2. `README.md` — documentation
  3. `steering-examples.json` — steering events for orchestration

**ตัวอย่าง error**:
```
✗ missing: managed-agent-cookbooks/kyc-screener/steering-examples.json
```

## Output Format

### Success Case

```
$ python3 scripts/check.py
OK — 127 file(s) checked, 0 issues.
```

Exit code: `0`

### Failure Case

```
$ python3 scripts/check.py
FAIL — 3 issue(s) across 120 file(s):

  ✗ YAML parse: managed-agent-cookbooks/pitch-agent/agent.yaml: mapping values are not allowed here (line 12)
  ✗ ref: managed-agent-cookbooks/gl-reconciler/agent.yaml: system.file -> system.txt (not found)
  ✗ missing: managed-agent-cookbooks/kyc-screener/steering-examples.json
```

Exit codes:
- `0` — ไม่พบ error
- `1` — พบ error (แสดงรายการหมด)
- `2` — Missing pyyaml library

## ตัวอย่าง (Example Workflows)

### Example 1: Quick Validation Before Commit

```bash
$ python3 scripts/check.py
OK — 127 file(s) checked, 0 issues.

$ git commit -m "Add new earnings-reviewer agent"
```

### Example 2: Detecting Broken References

```bash
$ # Rename subagent file
mv ./managed-agent-cookbooks/pitch-agent/subagents/researcher.yaml \
   ./managed-agent-cookbooks/pitch-agent/subagents/researcher_v2.yaml

$ python3 scripts/check.py
FAIL — 1 issue(s) across 127 file(s):

  ✗ ref: managed-agent-cookbooks/pitch-agent/agent.yaml: callable_agents.manifest -> ./subagents/researcher.yaml (not found)

$ # Fix by updating agent.yaml
vi ./managed-agent-cookbooks/pitch-agent/agent.yaml  # Change path to researcher_v2.yaml

$ python3 scripts/check.py
OK — 127 file(s) checked, 0 issues.
```

### Example 3: Bundled Skills Drift

```bash
$ # Edit a skill in vertical-plugins
vi ./plugins/vertical-plugins/tearsheet/skill.md

$ python3 scripts/check.py
FAIL — 1 issue(s) across 127 file(s):

  ✗ bundled-skill: plugins/agent-plugins/pitch-agent/skills/tearsheet: drifted from plugins/vertical-plugins/tearsheet (run scripts/sync-agent-skills.py)

$ # Sync skills
python3 scripts/sync-agent-skills.py
Syncing: tearsheet
  From: plugins/vertical-plugins/tearsheet
  To:   plugins/agent-plugins/pitch-agent/skills/tearsheet
  ✓ Updated 3 files

$ python3 scripts/check.py
OK — 127 file(s) checked, 0 issues.
```

### Example 4: Missing Required File

```bash
$ # Accidentally delete steering-examples.json
rm ./managed-agent-cookbooks/earnings-reviewer/steering-examples.json

$ python3 scripts/check.py
FAIL — 1 issue(s) across 127 file(s):

  ✗ missing: managed-agent-cookbooks/earnings-reviewer/steering-examples.json

$ # Restore from template or copy from similar agent
cp ./managed-agent-cookbooks/pitch-agent/steering-examples.json \
   ./managed-agent-cookbooks/earnings-reviewer/steering-examples.json

$ # Edit to match earnings-reviewer's actual steering events
vi ./managed-agent-cookbooks/earnings-reviewer/steering-examples.json

$ python3 scripts/check.py
OK — 127 file(s) checked, 0 issues.
```

## Check Functions — Internal Reference

| Function | Purpose | Triggered By |
|----------|---------|---|
| `check_refs(yml)` | Resolve system.file, skills, callable_agents paths | Step 4 |
| `yaml.safe_load()` | Parse YAML safely | Step 1 |
| `json.loads()` | Parse JSON | Step 2 |
| `filecmp.dircmp()` | Compare bundled skills with source | Step 4b |
| `re.findall()` | Extract skill references from markdown | Step 4b2 |

## Integration with CI/CD Pipeline

Typical workflow:

```bash
# .github/workflows/validate.yml
name: Validate Manifests
on: [push, pull_request]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      - run: pip install pyyaml
      - run: python scripts/check.py  # Fails if exit code != 0
```

## เกี่ยวข้องกับอะไรอีกบ้าง

| Script | ความสัมพันธ์ |
|--------|---|
| **sync-agent-skills.py** | ถ้าพบ "bundled-skill" error ให้รัน script นี้เพื่อ sync |
| **deploy-managed-agent.sh** | ควร run `check.py` ก่อนการ deploy เพื่อให้ manifest พร้อม |
| **CI/CD workflow** | โดยปกติเรียก `check.py` ในลำดับแรกของ validation pipeline |

## Exit Code Reference

```python
# From check.py (บรรทัด 160-166)
if errors:
    print(f"FAIL — {len(errors)} issue(s) across {checked} file(s):\n", file=sys.stderr)
    for e in errors:
        print(f"  ✗ {e}", file=sys.stderr)
    sys.exit(1)
else:
    print(f"OK — {checked} file(s) checked, 0 issues.")
    sys.exit(0)
```

---

**หมายเหตุ**: Script ถูกออกแบบให้ run ก่อน deploy เพื่อป้องกันการ deploy agent ที่มี manifest errors
