---
title: Scripts Reference — 6 Automation Tools
parent: 02 โครงสร้างโฟลเดอร์
nav_order: 6
---

# Scripts Overview

ที่อยู่: `scripts/`

Repository มี **6 scripts** ที่ทำงาน complementary — บาง scripts ทำงาน locally ก่อน commit, บางตัวทำงาน CI, บางตัว deployment-time

## 1. check.py — Pre-commit Linting

```bash
python3 scripts/check.py
```

### Purpose
Lint และ validate **ทุก** manifest ไฟล์ — ก่อนผู้ใช้รูป push

### ตรวจสอบ

| Check | ตรวจหา | ทำไม |
|---|---|---|
| YAML parse | ทุก `.yaml` ไฟล์ parses | ป้องกัน syntax errors |
| JSON parse | `plugin.json`, `marketplace.json`, `steering-examples.json` | Valid JSON structure |
| Frontmatter | `.md` files มี valid YAML frontmatter | Agent system prompts valid |
| File references | `system.file`, `skills[].path`, `callable_agents[].manifest` resolve | ไม่มี broken paths |
| Required files | ทุก `managed-agent-cookbooks/<slug>/` มี `agent.yaml`, `README.md`, `steering-examples.json` | Complete cookbooks |
| Skill drift | Bundled skill ใน `agent-plugins/` ตรงกับ source ใน `vertical-plugins/` | Sync ไม่ออกแบบ |

### Exit Code
- `0` = all checks pass ✓
- `1` = failures found, details printed ✗

### Usage Pattern

```bash
# Developer workflow
$ python3 scripts/check.py
✓ Checked 50+ manifests, 200+ skill references
✓ All bundled skills in sync
$ git commit
$ git push
```

## 2. sync-agent-skills.py — Propagate Skill Updates

```bash
python3 scripts/sync-agent-skills.py
```

### Purpose
Keep bundled skill copies ใน **agent-plugins** synchronized กับ source ใน **vertical-plugins**

### Mechanism

```
vertical-plugins/financial-analysis/skills/comps-analysis/
    ↓ (is source of truth)
agent-plugins/pitch-agent/skills/comps-analysis/
agent-plugins/market-researcher/skills/comps-analysis/
agent-plugins/model-builder/skills/comps-analysis/
    ↓ (all updated)
```

### Implementation

1. Scans `vertical-plugins/*/skills/*/` → index all skills by name
2. For each `agent-plugins/*/skills/*/`:
   - Find matching vertical source by name
   - `rm -rf` old bundled copy
   - `cp -r` fresh copy from source
3. Report: `synced 15 bundled skill dir(s) from vertical-plugins/`

### When to Run

- **After editing any skill ใน `vertical-plugins/`**
  
  Example: Fix comps calculation bug ใน `vertical-plugins/financial-analysis/skills/comps-analysis/`
  ```bash
  # Edit file
  vim plugins/vertical-plugins/financial-analysis/skills/comps-analysis/SKILL.md
  
  # Run sync
  python3 scripts/sync-agent-skills.py
  
  # Now all 3 agents that use comps-analysis are updated
  ```

### Exit Code
- `0` = synced successfully
- `1` = found bundled skills with no vertical source (error in repo structure)

## 3. deploy-managed-agent.sh — API Deployment

```bash
export ANTHROPIC_API_KEY=sk-ant-...
scripts/deploy-managed-agent.sh gl-reconciler
```

### Purpose
Deploy a **single managed agent** to `/v1/agents` API

### Steps

1. **Read manifest**: `managed-agent-cookbooks/gl-reconciler/agent.yaml`
2. **Resolve conveniences**: 
   - Inline system prompts from `.md` files
   - Upload skills ไป `/v1/skills`
   - Create subagents ไป `/v1/agents`
3. **Build final payload**: Valid JSON for `/v1/agents` POST
4. **POST to API**: Create agent, return `agent_id`

### Requires

```bash
# Environment
export ANTHROPIC_API_KEY=sk-ant-...       # Required unless --dry-run

# Tools
command -v jq                               # JSON processor
command -v python3                          # Python 3
python3 -c 'import yaml'                    # PyYAML module
```

### Manifest Transform Example

**Input** (agent.yaml):
```yaml
system:
  file: ../../plugins/agent-plugins/gl-reconciler/agents/gl-reconciler.md
  append: |
    Remember: all GL breaks must be traced to source
```

**Output** (resolved POST body):
```json
{
  "system": "You are a GL Reconciler assistant... [inlined content] Remember: all GL breaks must be traced to source"
}
```

### Options

```bash
scripts/deploy-managed-agent.sh gl-reconciler         # Deploy to production API
scripts/deploy-managed-agent.sh gl-reconciler --dry-run  # Show what would be POSTed, don't actually deploy
```

### Exit Code
- `0` = deployed successfully
- `1` = manifest file not found / API error / validation failed

## 4. test-cookbooks.sh — Dry-run All Agents

```bash
bash scripts/test-cookbooks.sh
```

### Purpose
Validate **ทุก** managed agent cookbook ว่า resolvable + well-formed โดยไม่ deploy

### Checks

ทุก `managed-agent-cookbooks/<slug>/agent.yaml`:
1. Parses YAML ✓
2. Resolves manifest conveniences ✓
3. Output valid JSON ✓
4. Depth ≤ 1 (no deep nesting of subagents) ✓
5. Non-empty system prompt ✓
6. No `output_schema` ใน orchestrator agent (only leaf workers มี) ✓

### Exit Code
- `0` = all cookbooks valid
- `1` = any cookbook fails

## 5. validate.py — Schema Validation for Outputs

```bash
python3 scripts/validate.py output.json schema.yaml
```

### Purpose
Validate **JSON output** from leaf worker agents against their declared `output_schema`

### Use Case

Reader subagent (e.g. `transcript-reader`) outputs JSON:
```json
{
  "ticker": "AAPL",
  "earnings_per_share": 6.05,
  "guidance": "10-15% growth FY2024"
}
```

Orchestrator validates ก่อนส่งไปยัง next agent:
```bash
python3 scripts/validate.py transcript.json \
  managed-agent-cookbooks/earnings-reviewer/subagents/transcript-reader-output-schema.yaml
```

If invalid → exit 1, orchestrator rolls back (handles via event bus)

### Supports

- JSON schema + YAML schema
- Loads from `.json` or `.yaml` files
- Reports detailed error path if validation fails

## 6. orchestrate.py — Reference Event Loop

```bash
python3 scripts/orchestrate.py
```

### Purpose
**Reference implementation** of cross-agent handoff orchestration

**Important**: This is NOT for production — replace with Temporal, Airflow, Guidewire event bus

### What It Does

```
Loop:
1. Wait for agent output
2. Detect: does output contain {type: "handoff_request", ...}?
3. If yes:
   a. Extract target_agent, event, context_ref
   b. Schema-validate payload
   c. Check target_agent ใน ALLOWED_TARGETS
   d. Emit steering event to target agent
   e. Collect result
   f. Return result ไป originating agent
```

### Security

```python
ALLOWED_TARGETS = {
    "pitch-agent", "market-researcher", "earnings-reviewer", 
    "meeting-prep-agent", "model-builder", "gl-reconciler", 
    "kyc-screener", "valuation-reviewer", "month-end-closer", 
    "statement-auditor",
}

HANDOFF_PAYLOAD_SCHEMA = {
    "type": "object",
    "additionalProperties": False,
    "required": ["event"],
    "properties": {
        "event": {"type": "string", "maxLength": 2000},
        "context_ref": {"type": "string", "maxLength": 256, ...}
    }
}
```

**Threat model**: Untrusted document reader could embed literal handoff JSON → mitigation: validate + allowlist

---

## Run Order During Development

```
Developer makes changes:
1. python3 scripts/check.py                    ← Validate all manifests
2. python3 scripts/sync-agent-skills.py        ← Sync bundled skills (if edited any)
3. [optional] python3 scripts/validate.py ...  ← Test specific output
4. bash scripts/test-cookbooks.sh              ← Dry-run all agents
5. git commit                                  ← Safe to commit
6. git push
```

## Run Order During CI

`.github/workflows/secret-scan.yml` (no agent build):
```
On PR / push to main:
→ Gitleaks check
→ Internal reference scrub
→ [no other CI — validation is local responsibility]
```

## Run Order During Deployment

```
Platform engineer:
1. bash scripts/test-cookbooks.sh --dry-run    ← Pre-flight check
2. export ANTHROPIC_API_KEY=...
3. scripts/deploy-managed-agent.sh pitch-agent ← Deploy 1 agent
4. scripts/deploy-managed-agent.sh gl-reconciler
5. [... repeat for each agent needed]
6. python3 scripts/orchestrate.py              ← Start event loop
```

---

**สรุป**: 
- `check.py` — pre-commit validator
- `sync-agent-skills.py` — propagate vertical sources ไป bundled copies
- `deploy-managed-agent.sh` — POST /v1/agents
- `test-cookbooks.sh` — dry-run all agents
- `validate.py` — schema checker for leaf outputs
- `orchestrate.py` — reference handoff event loop
