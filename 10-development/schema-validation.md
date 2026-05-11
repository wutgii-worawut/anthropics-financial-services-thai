---
title: Schema Validation
parent: 10 การ Development
nav_order: 3
---

# Schema Validation — validate.py

## validate.py ทำอะไร

Script `scripts/validate.py` ตรวจสอบว่า subagent output ตรงกับ JSON schema — เลย orchestrator ไม่ได้ consume malformed data

Usage:

```bash
python3 scripts/validate.py output.json schema.json
```

Exit 0 = valid, exit 1 = invalid

## Scenario: GL Reader Output

GL Reconciler agent — leaf worker `reader` อ่านจาก untrusted GL statement document

Output ต้องตรงกับ schema นี้:

```json
{
  "type": "object",
  "properties": {
    "breaks": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "trade_date": { "type": "string", "format": "date" },
          "break_amount": { "type": "number" },
          "description": { "type": "string", "maxLength": 500 }
        },
        "required": ["trade_date", "break_amount"]
      }
    }
  },
  "required": ["breaks"]
}
```

Reader output:

```json
{
  "breaks": [
    {
      "trade_date": "2026-05-10",
      "break_amount": 150000.50,
      "description": "FX conversion mismatch"
    }
  ]
}
```

Validate:

```bash
python3 scripts/validate.py reader-output.json schema.json
# OK
```

Invalid output:

```json
{
  "breaks": [
    {
      "trade_date": "2026-05-10",
      "break_amount": "150000.50"    # ← ควรเป็น number, ไม่ string!
    }
  ]
}
```

```bash
python3 scripts/validate.py reader-output.json schema.json
# INVALID: '150000.50' is not of type 'number' at /breaks/0/break_amount
```

## Define Schema ใน Subagent Manifest

Subagent yaml มี `output_schema` block:

```yaml
# managed-agent-cookbooks/gl-reconciler/subagents/reader.yaml

name: reader
model: claude-opus-4-7

system:
  text: "Parse GL statement. Return JSON only."

tools:
  - type: agent_toolset_20260401
    default_config: { enabled: false }
    configs:
      - name: read
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
          description: { type: string, maxLength: 500 }
        required: ["trade_date", "break_amount"]
  required: ["breaks"]
```

## Deploy Script Integration

`scripts/deploy-managed-agent.sh` extract `output_schema` จาก subagent yaml:

```bash
# ใน deploy-managed-agent.sh
json=$(jq '.output_schema as $schema | ...' agent.yaml)
# เก็บ $schema ไว้

# หลังจาก worker complete:
python3 scripts/validate.py worker-output.json <extracted-schema>
if [[ $? -ne 0 ]]; then
  # route error event back to worker
  orchestrator.emit("validation_failed", worker_id=..., output=...)
fi
```

## Schema ไฟล์ Type Formats

validate.py รองรับ YAML และ JSON:

```bash
# YAML schema
python3 scripts/validate.py output.json schema.yaml
# ✓ reads both formats

# JSON schema
python3 scripts/validate.py output.json schema.json
# ✓ works too
```

## Common Schema Constraints

Prevent untrusted input from reaching downstream:

### String Max Length

```yaml
description:
  type: string
  maxLength: 500    # ← cap length so no DOS
```

### Enum-only Values

```yaml
status:
  type: string
  enum: ["VERIFIED", "UNVERIFIED"]    # ← whitelist only
```

### Array Max Items

```yaml
breaks:
  type: array
  maxItems: 1000    # ← prevent memory explosion
```

### Numeric Bounds

```yaml
break_amount:
  type: number
  minimum: -1e9
  maximum: 1e9      # ← reasonable bounds
```

### No Additional Properties

```yaml
type: object
additionalProperties: false    # ← strict, no surprise fields
```

## Testing Output Schema

Before deploying subagent:

```bash
# Generate test output (pretend output from worker)
cat > test-reader-output.json << 'EOF'
{
  "breaks": [
    {
      "trade_date": "2026-05-10",
      "break_amount": 150000.50
    }
  ]
}
EOF

# Copy schema from subagent manifest
python3 scripts/validate.py test-reader-output.json \
  managed-agent-cookbooks/gl-reconciler/subagents/reader-schema.json

# OK → proceed, else fix worker system prompt
```

## Validation in Orchestrator Loop

Reference implementation `scripts/orchestrate.py` validates worker output:

```python
import subprocess
import json

def validate_worker_output(worker_name, output, schema_path):
    """Validate worker output against schema."""
    # Write temp output
    with open("/tmp/worker_output.json", "w") as f:
        json.dump(output, f)
    
    # Run validate.py
    result = subprocess.run(
        ["python3", "scripts/validate.py", 
         "/tmp/worker_output.json", schema_path],
        capture_output=True
    )
    
    if result.returncode != 0:
        # Validation failed
        error_msg = result.stderr.decode()
        emit_event("validation_failed", {
            "worker": worker_name,
            "error": error_msg,
            "output": output
        })
        return False
    
    return True
```

## Error Message Format

Validation error ชัดเจน:

```
INVALID: 'string_value' is not of type 'number' 
at /breaks/0/break_amount
```

Components:

- `'string_value'` — actual value
- `is not of type 'number'` — schema constraint
- `/breaks/0/break_amount` — path in JSON tree

## Disable Validation (Dev Only)

ในช่วง development ถ้า worker output ยังไม่ stable:

```yaml
# subagent.yaml
output_schema: null    # disable validation temporarily
```

Or in orchestrator:

```python
if os.getenv("SKIP_VALIDATION"):
    return True  # skip check
```

**But before production deployment — enable validation สำหรับ security ทั้งหมด**

---

**Next:** [Security CI — secret scanning ใน CI/CD](security-ci.md)
