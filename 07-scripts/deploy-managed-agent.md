---
title: deploy-managed-agent.sh
parent: 07 Scripts
nav_order: 5
---

# deploy-managed-agent.sh — Deploy Managed Agent ไป Anthropic API

## หน้าที่ (Purpose)

Script นี้คือ main deployment tool สำหรับ managed agents บนคลาวด์ Anthropic มันใช้ manifest file (agent.yaml) และ resolve ทั้งหมด ชิ้นส่วน อาทิ system prompt, skills, subagents แล้วส่ง POST request ไป `/v1/agents` API

**เมื่อควรใช้:**
- Deploy agent ใหม่ ไปคลาวด์
- Update agent ที่มีอยู่เดิม
- ในขั้นตอน CI/CD automation
- Test manifest ด้วย `--dry-run` flag ก่อน production deploy

## Usage

```bash
scripts/deploy-managed-agent.sh <slug>
scripts/deploy-managed-agent.sh <slug> --dry-run
```

**Arguments:**
- `<slug>` (required): ชื่อของ agent (folder name ใน `managed-agent-cookbooks/`)
- `--dry-run` (optional): ดูว่า manifest resolve เป็นอะไร โดยไม่ POST ไปที่ API

**Environment variables:**
- `ANTHROPIC_API_KEY` (required): API key สำหรับ authentication
- `ANTHROPIC_API_BASE` (optional): custom API endpoint (default: `https://api.anthropic.com`)
- `SKILL_TITLE_PREFIX` (optional): prefix สำหรับชื่อ skill (default: none)
- `DEPLOY_DEBUG` (optional): print debug info ระหว่าง resolve

**ข้อกำหนด:**
- `bash` 4.0+
- `jq` (JSON processing)
- `python3` + `pyyaml`
- `curl`
- `zip` (สำหรับ skill upload)

**รันจาก:** Repository root

## ภายในทำอะไร (How It Works)

### 1. Manifest Resolution (บรรทัด 88-112)

ฟังก์ชัน `resolve_manifest()` ดำเนินการ:

**a) Expand from_plugin directive:**
หากมี `skills[].from_plugin: "path/to/plugin"` script จะ expand เป็น `skills[].path` สำหรับทุก subdirectory ใน `plugin/skills/`

```yaml
# ใน agent.yaml
skills:
  - from_plugin: ../../plugins/agent-plugins/pitch-agent
    
# จะ expand เป็น:
skills:
  - path: ../../plugins/agent-plugins/pitch-agent/skills/fundamental-analysis
  - path: ../../plugins/agent-plugins/pitch-agent/skills/technical-analysis
  # ... เป็นต้น
```

**b) Mark skills for upload:**
```bash
.skills = ((.skills // []) | map(
  if .path then {__upload: ($base + "/" + .path)}
  elif .__upload then .
  else . end))
```

แต่ละ skill path จะถูก mark ด้วย `__upload` key เพื่อ distinguish จากอื่น

### 2. Skill Upload (บรรทัด 54-86)

ฟังก์ชัน `upload_skill()` อัพโหลด skill zip ไปยัง `/v1/skills` API:

```bash
zip="$(mktemp -t skill).zip"
(cd "$(dirname "$path")" && zip -qr "$zip" "$(basename "$path")")
resp=$(curl -sS "$API/v1/skills" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-beta: skills-2025-10-02" \
  -F "display_title=${SKILL_TITLE_PREFIX:-}$(basename "$path")" \
  -F "files[]=@$zip")
id=$(jq -r '.id // empty' <<<"$resp")
```

Script เก็บ cache เพื่อไม่ให้ upload skill ซ้ำสำหรับ agent เดียว:
- ถ้า skill มี cache แล้ว ใช้ cached skill_id
- ถ้า dry-run → ใช้ dummy `DRYRUN_<name>` ID

### 3. System Prompt Inline (บรรทัด 114-130)

ฟังก์ชัน `inline_system()` convert manifest system block:

```yaml
# ใน agent.yaml
system:
  file: system-prompt.md
  append: "Additional instructions here"
```

เป็น string ใน JSON:
```bash
sysfile=$(jq -r '.system.file // empty' <<<"$json")
text=$(jq -r '.system.text // empty' <<<"$json")
append=$(jq -r '.system.append // empty' <<<"$json")
body="$text"
if [[ -n "$sysfile" ]]; then
  body="$(cat "$base/$sysfile")"
fi
[[ -n "$append" ]] && body="${body}"$'\n\n'"${append}"
jq --arg s "$body" '.system=$s' <<<"$json"
```

### 4. Recursive Agent Creation (บรรทัด 132-171)

ฟังก์ชัน `create_agent()` ทำ recursive creation:

```bash
# สำหรับแต่ละ callable_agents[].manifest
for m in callable_agents[].manifest:
    # สร้าง subagent ก่อน (recursive call)
    out=$(create_agent "$base/$m")
    sid=${out%% *}  # agent_id
    sver=${out##* }  # version
    # add to callable_agents list
    sub_ids=$(jq --arg i "$sid" --argjson v "$sver" '. + [{type:"agent", id:$i, version:$v}]' <<<"$sub_ids")
```

หลังจากสร้างทั้ง skills และ subagents post ไปยัง API:

```bash
resp=$(req -X POST "$API/v1/agents" -d "$json")
id=$(jq -r '.id // empty' <<<"$resp")
```

### 5. Dry-Run Mode (บรรทัด 173-179)

ถ้า `--dry-run`:
- ไม่ post ไปที่ API
- เก็บ resolved JSON bodies ทั้งหมด (subagents first, orchestrator last)
- พิมพ์ out เป็น JSON array:

```bash
jq -s '.' "$DRY_OUT"  # Combine all bodies into array
```

## Output

**Normal deploy:**
```
deployed: pitch-agent
agent id: agent-123abc
console:  https://console.anthropic.com/agents/agent-123abc
```

**Dry-run:**
```
# --dry-run: resolved POST /v1/agents bodies (subagents first, orchestrator last)
[
  {
    "name": "earnings-extractor",
    "system": "Extract financial data from documents...",
    "model": "claude-opus-4-1-20250805",
    "skills": [...]
  },
  {
    "name": "pitch-agent",
    "system": "Create investment pitches based on...",
    "model": "claude-opus-4-1-20250805",
    "callable_agents": [
      {"type": "agent", "id": "agent-abc123", "version": 1}
    ]
  }
]
```

**Error cases:**
```
no manifest at /path/to/managed-agent-cookbooks/unknown-agent/agent.yaml
requires jq
requires python3 + pyyaml
POST /v1/skills failed for fundamental-analysis:
{"error": "API error..."}
POST /v1/agents failed for pitch-agent:
{"error": "API error..."}
```

**Exit codes:**
- `0` — deploy สำเร็จ
- `1` — manifest ไม่พบ, missing tools, หรือ API error

## ตัวอย่าง (Example)

### Example 1: Deploy Orchestrator with 2 Subagents

```bash
# ดูว่า manifest resolve เป็นอะไร ก่อน
$ bash scripts/deploy-managed-agent.sh pitch-agent --dry-run
# --dry-run: resolved POST /v1/agents bodies (subagents first, orchestrator last)
# [
#   { "name": "earnings-extractor", ... },
#   { "name": "market-researcher", ... },
#   { "name": "pitch-agent", "callable_agents": [...], ... }
# ]

# ดูดี → deploy เลย
$ export ANTHROPIC_API_KEY="sk-..."
$ bash scripts/deploy-managed-agent.sh pitch-agent
deployed: pitch-agent
agent id: agent-123abc
console:  https://console.anthropic.com/agents/agent-123abc
```

### Example 2: Deploy Multiple Agents

```bash
# Deploy earnings reviewer
$ bash scripts/deploy-managed-agent.sh earnings-reviewer
deployed: earnings-reviewer
agent id: agent-456def

# Deploy GL reconciler
$ bash scripts/deploy-managed-agent.sh gl-reconciler
deployed: gl-reconciler
agent id: agent-789ghi

# Deploy month-end closer (orchestrator ที่เรียก 2 agents ข้างบน)
$ bash scripts/deploy-managed-agent.sh month-end-closer
deployed: month-end-closer
agent id: agent-xyz123
```

### Example 3: Debug Skill Upload

```bash
$ export DEPLOY_DEBUG=1
$ bash scripts/deploy-managed-agent.sh pitch-agent 2>&1 | grep -E "callable|__upload"
{"name":"earnings-extractor","callable_agents":[]}
{"name":"market-researcher","callable_agents":[]}
{"name":"pitch-agent","callable_agents":[{"type":"agent","id":"agent-456def","version":1}]}
```

### Example 4: Environment Variable Substitution

```bash
# ใน agent.yaml อาจมี:
system:
  text: |
    You are an expert in ${DOMAIN}
    Report to team: ${REPORT_EMAIL}

# Deploy ด้วย env vars
$ export DOMAIN="Financial Services"
$ export REPORT_EMAIL="analysts@firm.com"
$ bash scripts/deploy-managed-agent.sh specialized-agent
```

## เกี่ยวข้องกับอะไรอีกบ้าง

- **check.py**: ต้องรัน ก่อน `deploy-managed-agent.sh` เพื่อให้ manifest พร้อม
- **test-cookbooks.sh**: ใช้ `deploy-managed-agent.sh --dry-run` เพื่อตรวจสอบทุก agent
- **orchestrate.py**: รับ agent_id ที่ได้จาก deploy นี้
- **validate.py**: เมื่อ subagent มี `output_schema` ในไฟล์ manifest
- **Anthropic Managed Agents API**: POST `/v1/agents`, `/v1/skills`

## Manifest Schema Reference

```yaml
name: "pitch-agent"
description: "Creates investment pitches"
model: "claude-opus-4-1-20250805"

# System prompt (one of: file, text, or both with append)
system:
  file: "system-prompt.md"
  append: "Append additional instructions here"

# Skills
skills:
  - path: "skills/fundamental-analysis"
  - path: "skills/technical-analysis"
  - from_plugin: "../../plugins/agent-plugins/equities-plugin"  # expands to all skills/

# Callable subagents
callable_agents:
  - manifest: "earnings-extractor/agent.yaml"
  - manifest: "market-researcher/agent.yaml"

# For reader subagents: schema validation
output_schema:
  type: "object"
  required: ["ticker", "price_target"]
  properties:
    ticker: { type: "string" }
    price_target: { type: "number" }
```
