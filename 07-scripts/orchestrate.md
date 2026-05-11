---
title: orchestrate.py
parent: 07 Scripts
nav_order: 2
---

# orchestrate.py — Reference Event Loop สำหรับ Cross-Agent Handoffs

## หน้าที่ (Purpose)

Script นี้เป็น reference implementation ของ event loop ที่ catch handoff requests ระหว่าง managed agents เช่น agent A อาจขอให้ agent B ทำการประเมินค่า (valuation) โดยการส่ง handoff message ผ่าน output text

**⚠️ หมายเหตุ:** นี่คือ reference code ที่ใช้เพื่อแสดง pattern เท่านั้น ในผลิตชั้นจริง ควรแทนที่ด้วย workflow engine ที่เหมาะสม เช่น Temporal, Airflow, หรือ Guidewire event bus

**เมื่อควรใช้:**
- เข้าใจวิธี orchestration pattern ของ managed agents ทำงาน
- ทดสอบ handoff logic ในสภาวะ non-production
- สร้าง orchestrator ที่ปรับตัวตามความต้องการของบริษัทของคุณ

## Usage

```bash
python3 scripts/orchestrate.py
```

**Environment variables:**
- `SOURCE_SESSION_ID` (required): ID ของ agent session ที่เราต้องการ listen สำหรับ handoff requests
- `AGENT_IDS` (optional): JSON object ที่ map agent slug → deployed agent_id

```bash
export SOURCE_SESSION_ID="agent-sess-abc123def456"
export AGENT_IDS='{"pitch-agent":"agent-789xyz","earnings-reviewer":"agent-456uvw"}'
python3 scripts/orchestrate.py
```

**ข้อกำหนด:**
- Python 3.7+
- `anthropic` SDK (`pip install anthropic`)
- `jsonschema` library (`pip install jsonschema`)

## ภายในทำอะไร (How It Works)

### 1. Handoff Request Pattern (บรรทัด 40-42)
Script ค้นหา handoff request ที่ฝังอยู่ใน agent's text output ด้วย regex pattern:

```python
HANDOFF_RE = re.compile(r'\{"type":\s*"handoff_request".*?\}', re.DOTALL)
```

Handoff blob ต้องมี JSON object เช่น:
```json
{
  "type": "handoff_request",
  "target_agent": "earnings-reviewer",
  "payload": {
    "event": "Please analyze the Q4 earnings report",
    "context_ref": "doc/earnings-2025-q4.pdf"
  }
}
```

### 2. Security Allowlist (บรรทัด 23-27)
ก่อนส่ง handoff ไปยัง target agent script จะตรวจสอบว่า `target_agent` อยู่ใน hardcoded allowlist:

```python
ALLOWED_TARGETS = {
    "pitch-agent", "market-researcher", "earnings-reviewer", 
    "model-builder", "gl-reconciler", "kyc-screener",
    "valuation-reviewer", "month-end-closer", "statement-auditor",
}
```

หากเอา agent ใหม่เข้าระบบ ต้องเพิ่มชื่อใน ALLOWED_TARGETS

### 3. Schema Validation (บรรทัด 29-38)
Payload ต้องตรงกับ schema นี้:

```python
HANDOFF_PAYLOAD_SCHEMA = {
    "type": "object",
    "additionalProperties": False,
    "required": ["event"],
    "properties": {
        "event": {"type": "string", "maxLength": 2000},
        "context_ref": {"type": "string", "maxLength": 256,
                        "pattern": r"^[A-Za-z0-9 ._/:#-]+$"},
    },
}
```

- `event` ต้องเป็น string ≤ 2000 ตัวอักษร (required)
- `context_ref` เป็น optional reference ชี้ไปยังเอกสารหรือบริบท

### 4. Extract & Validate (บรรทัด 45-61)
ฟังก์ชัน `extract_handoff()` ดึง handoff จาก text ด้วยขั้นตอน:

```python
def extract_handoff(text: str) -> dict | None:
    m = HANDOFF_RE.search(text)           # Find the blob
    if not m: return None
    obj = json.loads(m.group(0))          # Parse JSON
    target = obj.get("target_agent")
    if target not in ALLOWED_TARGETS:     # Allowlist check
        return None
    jsonschema.validate(instance=payload, schema=HANDOFF_PAYLOAD_SCHEMA)  # Schema check
    return {"target_agent": target, "payload": payload}
```

### 5. Event Loop (บรรทัด 64-82)
ฟังก์ชัน `run()` คือ main orchestration loop:

```python
def run(source_session_id: str, agent_ids: dict[str, str]) -> None:
    client = anthropic.Anthropic()
    with client.beta.agents.sessions.stream(session_id=source_session_id) as stream:
        for event in stream:
            if event.type != "message_delta" or not getattr(event, "text", None):
                continue
            handoff = extract_handoff(event.text)
            if not handoff: continue
            target_id = agent_ids.get(handoff["target_agent"])
            if not target_id: continue
            client.beta.agents.sessions.steer(  # Steer the target agent
                agent_id=target_id,
                input=handoff["payload"]["event"],
            )
```

Loop นี้:
1. Stream message deltas จาก source session
2. ค้นหา handoff blob ในแต่ละ text chunk
3. Validate ด้วย allowlist + schema
4. ค้นหา target agent ID จาก `agent_ids` map
5. Steer (ส่ง input ไป) target agent session

## Output

Script ไม่ print อะไร — มันเพียงแค่ listen และ forward handoffs ไปยัง target agents ตามความเงียบ ๆ

หากเกิด error:
- Invalid JSON ในผลลัพธ์ของ agent: silently skipped
- Target agent not in allowlist: silently skipped
- Target agent_id not found: silently skipped
- Missing env var: Python exception จะ exit

## ตัวอย่าง (Example)

```bash
# Setup: deploy 2 agents first
$ bash scripts/deploy-managed-agent.sh pitch-agent
deployed: pitch-agent
agent id: agent-123abc

$ bash scripts/deploy-managed-agent.sh earnings-reviewer
deployed: earnings-reviewer
agent id: agent-456def

# Start a session with pitch agent
$ AGENT_ID="agent-123abc"
curl -X POST https://api.anthropic.com/v1/agents/$AGENT_ID/sessions \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -d '{}'
# Returns: {"id": "agent-sess-xyz789"}

# In another terminal, run orchestrator
$ export SOURCE_SESSION_ID="agent-sess-xyz789"
$ export AGENT_IDS='{"earnings-reviewer":"agent-456def"}'
$ python3 scripts/orchestrate.py
# (Now listening for handoffs...)

# Agent output contains:
# {"type":"handoff_request","target_agent":"earnings-reviewer","payload":{"event":"Analyze Q4 earnings"}}
# → orchestrate.py extracts it, validates it, then calls:
#   client.beta.agents.sessions.steer(agent_id="agent-456def", input="Analyze Q4 earnings")
```

## เกี่ยวข้องกับอะไรอีกบ้าง

- **deploy-managed-agent.sh**: Deploy agents ที่ต้องการจะ orchestrate
- **Managed Agents API** (`/v1/agents`): ใช้ session.stream() และ session.steer()
- **Workflow engines**: Temporal, Airflow ควรใช้ pattern นี้ เพื่อ implement in production

## Security Notes

- Handoff requests ที่ฝังในเอกสาร untrusted อาจถูก exploit ดังนั้น allowlist + schema validation เป็น minimum mitigation
- ในผลิตชั้นจริง ใช้ dedicated tool call หรือ typed SSE event แทน text parsing
- ตรวจสอบให้แน่ใจ AGENT_IDS env var ถูก set ถูกต้องสำหรับ prod
