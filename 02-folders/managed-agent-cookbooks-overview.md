---
title: Managed Agent Cookbooks — API Deployment Templates
parent: 02 โครงสร้างโฟลเดอร์
nav_order: 5
---

# Managed Agent Cookbooks Overview

ที่อยู่: `managed-agent-cookbooks/`

นี่คือ **mirror structure** ของ `plugins/agent-plugins/` — แต่สำหรับ **Claude Managed Agents API** deployment แทน Cowork

## ความสัมพันธ์ระหว่าง Cowork vs Managed Agents

| Aspect | Cowork Plugin | Managed Agent Cookbook |
|---|---|---|
| **Installation** | User clicks "Add plugin" | Platform engineer runs `deploy-managed-agent.sh` |
| **Manifest format** | `plugin.json` | `agent.yaml` (YAML + JSON) |
| **Deployment target** | Claude.com/product/cowork | `/v1/agents` API endpoint |
| **Invocation** | User types in chat | Steering event sent to `/messages` |
| **System prompt source** | Same `.md` file | Same `.md` file (copied) |
| **Skills** | Bundled files | Uploaded to `/v1/skills`, referenced by ID |
| **Subagents** | Not in Cowork | Depth-1 leaf workers with `callable_agents` |

**Key insight**: Agent logic เดียวกัน — Cowork และ API แค่ wrapper ต่างกัน

## Cookbook Structure

```
managed-agent-cookbooks/pitch-agent/
├── agent.yaml              # Main manifest (→ POST /v1/agents)
├── subagents/
│   ├── researcher.yaml     # Leaf worker 1
│   ├── modeler.yaml        # Leaf worker 2
│   └── deck-writer.yaml    # Leaf worker 3 (only one with Write)
├── steering-examples.json  # Example steering events
└── README.md               # Security tier + handoff notes
```

### 10 Cookbooks (1 per agent)

| Slug | Vertical Source | Leaf Workers | Bold (Write) |
|---|---|---|---|
| `pitch-agent` | investment-banking | researcher, modeler, **deck-writer** | deck-writer |
| `market-researcher` | equity-research | sector-reader, comps-spreader, **note-writer** | note-writer |
| `earnings-reviewer` | equity-research | transcript-reader, model-updater, **note-writer** | note-writer |
| `meeting-prep-agent` | wealth-management | profiler, news-reader, **pack-writer** | pack-writer |
| `model-builder` | financial-analysis | data-puller, **builder**, auditor | builder |
| `gl-reconciler` | financial-analysis | reader, critic, **resolver** | resolver |
| `kyc-screener` | financial-analysis | doc-reader, rules-engine, **escalator** | escalator |
| `valuation-reviewer` | private-equity | package-reader, valuation-runner, **publisher** | publisher |
| `month-end-closer` | financial-analysis | ledger-reader, rollforward, **poster** | poster |
| `statement-auditor` | private-equity | statement-reader, reconciler, **flagger** | flagger |

**Important**: ทุก agent มี 3 leaf workers, แต่เพียงตัวเดียวมี `Write` capability

## agent.yaml — Manifest Format

ตัวอย่างโครงสร้าง:

```yaml
name: pitch-agent
model: claude-3-5-sonnet-20241022
system:
  file: ../../plugins/agent-plugins/pitch-agent/agents/pitch-agent.md
  append: |
    ## Additional API-specific context
    You are running as a Managed Agent...

skills:
  - from_plugin: ../../plugins/agent-plugins/pitch-agent

callable_agents:
  - manifest: ./subagents/researcher.yaml
  - manifest: ./subagents/modeler.yaml
  - manifest: ./subagents/deck-writer.yaml
```

## Manifest Conveniences (Deploy Script Resolves)

Deploy script (`scripts/deploy-managed-agent.sh`) แปลง conveniences เหล่านี้:

| Manifest | Resolves to |
|---|---|
| `system: {file: ../../plugins/.../agents/pitch-agent.md, append: "..."}` | `system: "<inlined file content + append text>"` |
| `system: {text: "..."}` | `system: "<text>"` |
| `skills: [{from_plugin: ../../plugins/agent-plugins/pitch-agent}]` | uploads every skill under that dir → `[{type: custom, skill_id: ...}, ...]` |
| `skills: [{path: ../../...}]` | uploads skill → `{type: custom, skill_id: ...}` |
| `callable_agents: [{manifest: ./subagents/x.yaml}]` | creates agent first → `{type: agent, id: ..., version: latest}` |

### ตัวอย่าง Output

```json
{
  "name": "pitch-agent",
  "model": "claude-3-5-sonnet-20241022",
  "system": "You are Pitch Agent. You take competitor data... [inlined content from pitch-agent.md + append]",
  "skills": [
    {
      "type": "custom",
      "skill_id": "skill_abc123def456"  // uploaded from pitch-agent/skills/comps-analysis/
    },
    ...
  ],
  "callable_agents": [
    {
      "type": "agent",
      "id": "agent_researcher_uuid",
      "version": "latest"
    },
    ...
  ]
}
```

## Subagent Structure

ตัวอย่าง leaf worker:

```yaml
# subagents/researcher.yaml
name: pitch-agent-researcher
description: "Gathers market and company data for pitch book analysis"
model: claude-3-5-sonnet-20241022
system:
  text: |
    You are a research assistant for pitch book creation.
    Your job is to gather:
    - comparable company data
    - precedent transactions
    - market insights
    
    Return your findings as structured JSON.

skills:
  - path: ../../plugins/agent-plugins/pitch-agent/skills/comps-analysis
  - path: ../../plugins/agent-plugins/pitch-agent/skills/market-research

output_schema:
  type: object
  properties:
    comps:
      type: array
      items:
        type: object
        required: [ticker, ev, multiple]
```

### Output Schema Validation

Reader subagents (ไม่มี Write) อาจมี `output_schema:` block:
- Deploy script extracts schema
- `scripts/validate.py` checks output ก่อน orchestrator ประมวลผล
- ป้องกัน bad JSON ไม่ไปถึง next stage

**Bold/Write workers** ไม่มี schema (ดำเนินการเสร็จสิ้น)

## Steering Examples

ไฟล์ `steering-examples.json`:

```json
{
  "examples": [
    {
      "description": "Pitch book for acquisition scenario",
      "event": {
        "type": "steering_event",
        "target_agent": "pitch-agent",
        "payload": {
          "target": "Example Corp",
          "acquirer": "Strategic Buyer Inc",
          "thesis": "Synergies in cloud platform integration"
        }
      }
    }
  ]
}
```

**ใช้สำหรับ**: Documentation + orchestration layer reference

## Cross-Agent Handoffs

Agents ไม่เรียกกันโดยตรง — เมื่อ agent หนึ่งต้องการ agent อื่น:

```json
// Inside orchestrator output detection:
{
  "type": "handoff_request",
  "target_agent": "market-researcher",
  "event": "Get sector overview for TechCorp Inc",
  "context_ref": "pitch_deck_session_123"
}
```

**Flow**:
1. `pitch-agent` outputs text containing handoff JSON
2. `scripts/orchestrate.py` (or your event bus) detects
3. Schema validates: `target_agent` ใน allowlist?
4. Sends new steering event ไป `market-researcher`
5. Result returned back ไป `pitch-agent`

### Security

ข้อควรระวัง: ถ้าผู้ประชากร document สามารถ embed "handoff" text → อาจ trigger unintended handoff

**Mitigations** ใน `orchestrate.py`:
- Hard-allowlist target agents (ตัวอักษร set)
- Schema-validate payload (JSON structure)
- Prefer tool call over text pattern matching (ในอนาคต)

## README.md ในแต่ละ Cookbook

ตัวอย่าง:

```markdown
# pitch-agent Managed Agent

## Security Tier
Confidential — may process deal-sensitive materials

## Handoff Targets
- market-researcher: for deep sector analysis
- model-builder: for financial modeling

## Steering Event Schema
{
  "target": "Target Company Inc",
  "acquirer": "Buyer Group LLC",
  "thesis": "string describing deal thesis"
}
```

## Deploy Workflow

```bash
# Step 1: Check all cookbooks are valid
bash scripts/test-cookbooks.sh

# Step 2: Deploy specific agent
export ANTHROPIC_API_KEY=sk-ant-...
scripts/deploy-managed-agent.sh pitch-agent

# Output:
# - Creates 3 subagents (researcher, modeler, deck-writer)
# - Uploads all skills
# - POSTs orchestrator agent to /v1/agents
# - Returns agent_id for use in steering events
```

## ทำไมแยก Managed Agents ออกจาก Cowork?

| ด้าน | เหตุผล |
|---|---|
| **API Format** | `/v1/agents` expects YAML + JSON resolve steps (Cowork expects plugin.json) |
| **Subagents** | Cowork ไม่มี callable_agents (only agents, no workers) |
| **Steering** | API-driven (steering_event), Cowork user-driven (chat) |
| **Validation** | CMA needs schema checks ก่อน handoff (Cowork handles via UI) |
| **Orchestration** | CMA users manage event loop (e.g. Temporal, Airflow) |

---

**สรุป**: managed-agent-cookbooks/ เป็น API deployment mirror ของ plugins/agent-plugins/ — ใช้เดียว system prompts + skills แต่ wrap ใน `agent.yaml` manifests, subagent leaf workers, และ steering event orchestration ที่เหมาะสำหรับ headless deployment
