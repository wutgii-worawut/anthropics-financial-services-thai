---
title: test-cookbooks.sh
parent: 07 Scripts
nav_order: 6
---

# test-cookbooks.sh — Test All Agent Cookbooks สำหรับการ Deploy

## หน้าที่ (Purpose)

Script นี้รัน dry-run test บน **ทุก** managed agent cookbook ในระบบ เพื่อให้มั่นใจว่า:
1. Manifest ของแต่ละ agent resolve ได้สำเร็จ
2. Resolved POST bodies มีรูปแบบที่ถูกต้อง
3. ไม่มี system prompt ที่ว่างเปล่า
4. ไม่มี circular dependencies (subagent มี callable_agents nested อีกลำดับ)
5. `output_schema` ไม่ leak เข้าไปใน POST body

**เมื่อควรใช้:**
- ก่อน merge pull request ที่มีการเปลี่ยนแปลง manifest
- ในขั้นตอน CI/CD validation pipeline
- ตรวจสอบความสมบูรณ์ของทุก agent ก่อน bulk deploy

## Usage

```bash
bash scripts/test-cookbooks.sh
```

ไม่มี arguments — script scan ทั้งหมด `managed-agent-cookbooks/*/` และ test แต่ละอันโดยอัตโนมัติ

**ข้อกำหนด:**
- `bash` 4.0+
- `python3`
- `jq`
- `deploy-managed-agent.sh` (ใช้เป็น sub-script)

**รันจาก:** Repository root

## ภายในทำอะไร (How It Works)

### 1. Iterate All Cookbooks (บรรทัด 8-9)

```bash
for d in "$ROOT"/managed-agent-cookbooks/*/; do
  slug=$(basename "$d")
```

วนลูปแต่ละ subdirectory ใน `managed-agent-cookbooks/` และแยก folder name เป็น `slug`

### 2. Dry-Run Deploy (บรรทัด 10)

รัน `deploy-managed-agent.sh --dry-run` สำหรับแต่ละ agent:

```bash
bash "$ROOT/scripts/deploy-managed-agent.sh" "$slug" --dry-run 2>&1 | tail -n +2
```

- `--dry-run`: resolve manifest ไม่ post ไปที่ API
- `2>&1`: capture stderr ด้วย
- `tail -n +2`: skip header line "# --dry-run: resolved..."

### 3. JSON Validation in Python (บรรทัด 11-21)

ส่ง resolved bodies array ไปยัง Python script ที่ตรวจสอบ:

```python
import json,sys
b=json.load(sys.stdin)              # Load array of bodies
errs=[]

# Check 1: ทุก agent ต้องมี system prompt ไม่เป็นค่า falsy
for i,x in enumerate(b):
    if not x.get('system'): 
        errs.append(f'{x.get("name")}: empty system')
    
# Check 2: orchestrator ต้องอยู่ที่ level นอก subagent ไม่มี depth > 1
    if i<len(b)-1 and x.get('callable_agents'): 
        errs.append(f'{x.get("name")}: depth>1 (subagent has callable_agents)')

# Check 3: ไม่มี output_schema ใน final bodies (must be removed before API)
if 'output_schema' in json.dumps(b): 
    errs.append('output_schema leaked into a body')

if errs:
    for e in errs: print(f'      {e}', file=sys.stderr)
    sys.exit(1)
    
# Success
print(f'  ✓ {sys.argv[1]:24s} {len(b)} bodies')
```

### 4. Collect Results (บรรทัด 22-26)

หากผ่าน ✓ ถ้าไม่ผ่าน ✗ และ set `fail=1`:

```bash
if ! bash ... | python3 ... "$slug"; then
    echo "  ✗ $slug" >&2
    fail=1
fi
```

### 5. Exit Status (บรรทัด 27)

```bash
exit $fail
```

- `exit 0` ถ้าทั้งหมดผ่าน
- `exit 1` ถ้ามี agent fail

## Output

**ทั้งหมด pass:**
```
  ✓ pitch-agent             3 bodies
  ✓ earnings-reviewer       2 bodies
  ✓ gl-reconciler           1 body
  ✓ market-researcher       1 body
  ✓ valuation-reviewer      2 bodies
  ✓ month-end-closer        4 bodies
```

**มี agent fail:**
```
  ✓ pitch-agent             3 bodies
  ✗ earnings-reviewer
      earnings-extractor: empty system
      earnings-reviewer: depth>1 (subagent has callable_agents)
  ✓ gl-reconciler           1 body
  ✗ validate-test
      output_schema leaked into a body
```

Exit code: `1` (failure)

## ตัวอย่าง (Example)

### Example 1: All Cookbooks Pass

```bash
$ bash scripts/test-cookbooks.sh
  ✓ earnings-reviewer       2 bodies
  ✓ gl-reconciler           1 body
  ✓ kyc-screener            1 body
  ✓ market-researcher       1 body
  ✓ meeting-prep-agent      2 bodies
  ✓ model-builder           1 body
  ✓ month-end-closer        4 bodies
  ✓ pitch-agent             3 bodies
  ✓ statement-auditor       1 body
  ✓ valuation-reviewer      2 bodies

$ echo $?
0
```

### Example 2: Failure Detection

```bash
# ทำให้ pitch-agent ล้มเหลว โดยลบ system prompt
$ mv managed-agent-cookbooks/pitch-agent/system-prompt.md \
      managed-agent-cookbooks/pitch-agent/system-prompt.md.bak

$ bash scripts/test-cookbooks.sh
  ✓ earnings-reviewer       2 bodies
  ✓ gl-reconciler           1 body
  ✓ kyc-screener            1 body
  ✓ market-researcher       1 body
  ✓ meeting-prep-agent      2 bodies
  ✓ model-builder           1 body
  ✓ month-end-closer        4 bodies
  ✗ pitch-agent
      pitch-agent: empty system
  ✓ statement-auditor       1 body
  ✓ valuation-reviewer      2 bodies

$ echo $?
1

# Fix it
$ mv managed-agent-cookbooks/pitch-agent/system-prompt.md.bak \
      managed-agent-cookbooks/pitch-agent/system-prompt.md

$ bash scripts/test-cookbooks.sh
  ...
  ✓ pitch-agent             3 bodies
  ...
$ echo $?
0
```

### Example 3: Circular Dependency Test

```bash
# ถ้าแก้ไข pitch-agent manifest เพื่อมี callable_agents ระดับนอก
# (orchestrator level) และ pitch-agent เองมี callable_agents อีก:
$ bash scripts/test-cookbooks.sh
  ...
  ✗ pitch-agent
      earnings-extractor: depth>1 (subagent has callable_agents)
  ...
```

## Integration ใน CI/CD

### GitHub Actions Example

```yaml
name: Validate Cookbooks
on: [pull_request, push]

jobs:
  test-cookbooks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
      - run: sudo apt-get install -y jq
      - run: bash scripts/test-cookbooks.sh
```

### GitLab CI Example

```yaml
test:cookbooks:
  stage: validate
  image: ubuntu:latest
  before_script:
    - apt-get update && apt-get install -y jq python3
  script:
    - bash scripts/test-cookbooks.sh
```

## เกี่ยวข้องกับอะไรอีกบ้าง

- **deploy-managed-agent.sh**: script นี้เรียก `--dry-run` เพื่อ resolve ทุก manifest
- **check.py**: structural validation ของ manifest (syntax, refs)
- **validate.py**: schema validation สำหรับ agent output (ส่วนอื่น)
- **CI/CD pipeline**: `test-cookbooks.sh` ควรอยู่ก่อน actual deployment
- **Developer workflow**: 
  1. แก้ไข manifest (`agent.yaml`)
  2. รัน `bash scripts/test-cookbooks.sh` locally
  3. ถ้า pass → commit
  4. ถ้า fail → fix issues → re-test

## Troubleshooting

### "empty system" Error

```bash
# Check what's in system
$ jq '.system' managed-agent-cookbooks/my-agent/agent.yaml
null

# Fix: add system prompt
$ echo 'You are an expert agent.' > managed-agent-cookbooks/my-agent/system.txt
$ vim managed-agent-cookbooks/my-agent/agent.yaml
# เพิ่ม:
# system:
#   file: "system.txt"
```

### "output_schema leaked" Error

```bash
# ตรวจสอบว่าไม่มี output_schema ใน orchestrator level manifest
$ grep -r "output_schema:" managed-agent-cookbooks/*/agent.yaml

# output_schema ควรจะอยู่ใน subagent manifests เท่านั้น
# deploy-managed-agent.sh จะลบออกก่อน POST
```

### "depth>1" Error

```bash
# Orchestrator ต้องไม่มี callable_agents ซ้อนกันเกิน 1 ระดับ
# ถ้าต้องการ deep hierarchy ใช้ orchestrator chain แทน:
# Agent A → Orchestrator → [subagents]
# Orchestrator → Agent B → [subagents]
# (separate top-level agents)
```
