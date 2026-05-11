---
title: Managed Agents API
parent: 09 การ Deploy
nav_order: 3
---

# Managed Agents API — Deploy Agents ใน Production

## Managed Agents API คืออะไร

Managed Agents API (`POST /v1/agents`) ให้ deploy Claude agents headlessly ใน **your workflow** — Temporal, Airflow, Guidewire, หรือ custom event bus

ไม่ต่อ Cowork — คุณ own the orchestrator, คุณ own the steering event loop

## Agent Deployment ด้วย deploy-managed-agent.sh

Repository มี reference script ให้ upload skills, create leaf workers, และ POST orchestrator ไปยัง API:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
scripts/deploy-managed-agent.sh gl-reconciler
```

Output:

```
deployed: gl-reconciler
agent id: ag_xxxxxxxxxxxx
console:  https://console.anthropic.com/agents/ag_xxxxxxxxxxxx
```

## agent.yaml Manifest Schema

แต่ละ managed agent มี `agent.yaml` ที่กำหนด:

```yaml
name: gl-reconciler
model: claude-opus-4-7

system:
  file: ../../plugins/agent-plugins/gl-reconciler/agents/gl-reconciler.md
  append: "You are running headless. Produce files in ./out/; ..."

tools:
  - type: agent_toolset_20260401
    default_config: { enabled: false }
    configs:
      - name: read
        enabled: true
      - name: write
        enabled: false    # ← only resolver worker gets write
  - type: mcp_toolset
    mcp_server_name: internal-gl
    default_config: { enabled: true }

mcp_servers:
  - type: url
    name: internal-gl
    url: ${GL_MCP_URL}              # env var substitution

skills:
  - { from_plugin: ../../plugins/agent-plugins/gl-reconciler }

callable_agents:
  - manifest: ./subagents/reader.yaml
  - manifest: ./subagents/critic.yaml
  - manifest: ./subagents/resolver.yaml
```

| Field | Purpose |
|---|---|
| `name` | Agent display name (slug) |
| `model` | Claude model (opus-4-7, sonnet-4-7, ...) |
| `system` | System prompt — inline `{file: ...}` or raw `{text: ...}` |
| `tools` | Enable/disable standard toolset (read, write, edit, grep, glob, bash) |
| `mcp_servers` | Data connectors (read-only or full) |
| `skills` | Upload from `from_plugin` dir or explicit `path:` |
| `callable_agents` | Subagent manifests (depth-1 max) |

## Subagents (Leaf Workers)

GL Reconciler มี 3 leaf workers:

### reader.yaml — Reads untrusted documents

```yaml
name: reader
model: claude-opus-4-7
system:
  text: |
    You are a GL statement parser. Extract break lines from the document.
    Return JSON only, no prose.

tools:
  - type: agent_toolset_20260401
    default_config: { enabled: false }
    configs:
      - name: read
        enabled: true      # ← read only
      - name: grep
        enabled: true

output_schema:
  type: object
  properties:
    breaks:
      type: array
      items:
        type: object
        properties:
          trade_date: { type: string, format: date }
          break_amount: { type: number }
```

### critic.yaml — Verifies against trusted sources

```yaml
name: critic
model: claude-opus-4-7
system:
  text: |
    You are a GL break auditor. For each break, confirm it exists in the
    GL + subledger independently. Return VERIFIED or UNVERIFIED only.

tools:
  - type: agent_toolset_20260401
    default_config: { enabled: false }
    configs:
      - name: read
        enabled: true

mcp_servers:
  - type: url
    name: internal-gl
    url: ${GL_MCP_URL}    # read-only

output_schema:
  type: object
  properties:
    verified_breaks: { type: array }
    rejected_breaks: { type: array }
```

### resolver.yaml — Only worker with Write

```yaml
name: resolver
model: claude-opus-4-7
system:
  text: |
    You are the GL exception reporter. Take verified breaks and produce
    a sign-off report. Write to ./out/exception-report.md.

tools:
  - type: agent_toolset_20260401
    default_config: { enabled: false }
    configs:
      - name: read
        enabled: true
      - name: write
        enabled: true      # ← ONLY WORKER WITH WRITE
```

## Security Constraint: Depth-1 Only

Subagents cannot call further subagents. Max depth = 1 level.

```
Orchestrator (gl-reconciler)
├── reader           (no calls further)
├── critic           (no calls further)
└── resolver         (no calls further)
```

This prevents adversarial agent chains.

## POST /v1/agents — API Format

After deploy script resolves manifest, API call looks like:

```bash
curl -X POST https://api.anthropic.com/v1/agents \
  -H "x-api-key: sk-ant-..." \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: managed-agents-2026-04-01" \
  -H "content-type: application/json" \
  -d '{
    "name": "gl-reconciler",
    "model": "claude-opus-4-7",
    "system": "...",
    "tools": [
      {
        "type": "agent_toolset_20260401",
        "configs": [...]
      },
      {
        "type": "mcp_toolset",
        "mcp_server_name": "internal-gl"
      }
    ],
    "mcp_servers": [
      {
        "type": "url",
        "name": "internal-gl",
        "url": "https://gl.firm.internal/mcp"
      }
    ],
    "skills": [
      {
        "type": "custom",
        "skill_id": "sk_xxxxxxxx"
      }
    ],
    "callable_agents": [
      {
        "type": "agent",
        "id": "ag_reader_xxxx",
        "version": "latest"
      }
    ]
  }'
```

Response:

```json
{
  "id": "ag_gl_reconciler_xxxx",
  "version": 1,
  "created_at": "2026-05-12T10:30:00Z"
}
```

## Structured Output Validation

Subagents dengan `output_schema` block — deploy script wraps mereka:

```python
# deploy-managed-agent.sh detects output_schema
# เมื่อ subagent complete, script runs:

python3 scripts/validate.py \
  <worker-output.json> \
  <schema.json>
```

If validation fail → orchestrator rejects output, routes error event back to worker

## Steering Events & Orchestration

ลาด orchestrator ของ firm emit steering events:

```python
# Example: orchestrate.py reference implementation
event = {
  "agent_id": "ag_gl_reconciler_xxxx",
  "steering": {
    "task": "Reconcile GL vs subledger",
    "trade_date": "2026-05-10",
    "asset_classes": ["Equities", "FX"]
  }
}

response = client.agents.events.add(event)
# Agent starts. Returns session_id + stream of messages
```

Agent complete, emit output:

```json
{
  "status": "complete",
  "output": "...",
  "handoff_request": {
    "target_agent": "month-end-closer",
    "payload": {
      "entity": "North America",
      "period": "2026-05"
    }
  }
}
```

Your orchestrator check `handoff_request` — route payload ไปยัง `month-end-closer` agent

## Manifest Validation ก่อน Deploy

ก่อน `POST /v1/agents` run checks:

```bash
python3 scripts/check.py
```

Output:

```
✓ All manifests valid
✓ All references resolve
✓ No bundled skill drift detected
OK — 47 file(s) checked, 0 issues.
```

## Dry-run ก่อน Push to API

Test manifest resolution ไม่เสริมจริง:

```bash
scripts/deploy-managed-agent.sh gl-reconciler --dry-run
```

Output:

```
# --dry-run: resolved POST /v1/agents bodies (subagents first, orchestrator last)
[
  {
    "name": "reader",
    "model": "claude-opus-4-7",
    ...
  },
  {
    "name": "critic",
    ...
  },
  {
    "name": "resolver",
    ...
  },
  {
    "name": "gl-reconciler",
    "callable_agents": [
      {"type": "agent", "id": "ag_reader_xxxx", "version": "latest"},
      ...
    ]
  }
]
```

Verify JSON structure ก่อน deploy จริง

---

**Next:** [Development guide — validate, sync, customize](../10-development/manifest-validation.md)
