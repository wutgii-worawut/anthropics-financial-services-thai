---
title: validate.py
parent: 07 Scripts
nav_order: 4
---

# validate.py — Validate Managed Agent Output ตาม JSON Schema

## หน้าที่ (Purpose)

API ของ Managed Agents ไม่ได้ enforce structured output (JSON schema validation) จากฝั่ง API ดังนั้น script นี้เป็น validation wrapper ที่ตรวจสอบว่า output ของ subagent reader (เช่น "extract data from document") ตรงกับ schema ที่กำหนด ก่อนที่ orchestrator จะใช้มัน

**เมื่อควรใช้:**
- ในขั้นตอน CI/CD deploy เพื่อให้ subagent outputs ปลอดภัย
- เมื่อต้องการตรวจสอบ JSON schema validation ของ agent output
- ใน harness ระหว่าง reader subagent กับ orchestrator

## Usage

```bash
python3 scripts/validate.py <output.json> <schema.json|schema.yaml>
```

**Arguments:**
- `<output.json>`: ไฟล์ JSON ที่ต้องการตรวจสอบ (หรือ YAML)
- `<schema.json|schema.yaml>`: JSON Schema ไฟล์ (รองรับทั้ง .json และ .yaml)

**ข้อกำหนด:**
- Python 3.7+
- `jsonschema` library (`pip install jsonschema`)
- `pyyaml` (required if using .yaml schema)

**ตัวอย่าง:**
```bash
python3 scripts/validate.py earnings-data.json schema.json
python3 scripts/validate.py market-report.yaml validation-rules.yaml
```

## ภายในทำอะไร (How It Works)

### 1. Load Output (บรรทัด 18-23)
ฟังก์ชัน `_load()` รองรับทั้ง JSON และ YAML:

```python
def _load(path: Path):
    text = path.read_text()
    if path.suffix in (".yaml", ".yml"):
        import yaml
        return yaml.safe_load(text)
    return json.loads(text)
```

### 2. Load Schema (บรรทัด 26-31)
Main function เรียก `_load()` สำหรับทั้ง output และ schema:

```python
def main() -> int:
    if len(sys.argv) != 3:
        print(__doc__, file=sys.stderr)
        return 2
    instance = _load(Path(sys.argv[1]))  # Parse output
    schema = _load(Path(sys.argv[2]))    # Parse schema
```

### 3. Validate ด้วย jsonschema (บรรทัด 32-36)
ใช้ `jsonschema.validate()` เพื่อตรวจสอบ instance ตาม schema:

```python
try:
    jsonschema.validate(instance=instance, schema=schema)
except jsonschema.ValidationError as e:
    print(f"INVALID: {e.message} at {'/'.join(str(p) for p in e.absolute_path)}", file=sys.stderr)
    return 1
print("OK")
return 0
```

หากตรงกับ schema จะ print `OK` ถ้าไม่ตรง จะ print error message พร้อมตำแหน่ง path ในโครงสร้าง

## Output

**ถ้า valid:**
```
OK
```

**ถ้า invalid:**
```
INVALID: 'required' is a required property at /properties/earnings_per_share
```

Error message รวม:
- validation error message จาก jsonschema
- JSON path ที่ล้มเหลว (เช่น `/properties/name` หมายถึงที่ nested properties.name)

**Exit codes:**
- `0` — output ตรงกับ schema
- `1` — output ไม่ตรงกับ schema
- `2` — usage error (arguments ผิด)

## ตัวอย่าง (Example)

### Example 1: Valid Output

```bash
$ cat earnings-output.json
{
  "company": "ACME Corp",
  "eps": 2.50,
  "revenue_millions": 1500
}

$ cat schema.json
{
  "type": "object",
  "required": ["company", "eps", "revenue_millions"],
  "properties": {
    "company": {"type": "string"},
    "eps": {"type": "number"},
    "revenue_millions": {"type": "number"}
  }
}

$ python3 scripts/validate.py earnings-output.json schema.json
OK
$ echo $?
0
```

### Example 2: Missing Required Field

```bash
$ cat incomplete-output.json
{
  "company": "ACME Corp",
  "eps": 2.50
  # missing: revenue_millions
}

$ python3 scripts/validate.py incomplete-output.json schema.json
INVALID: 'revenue_millions' is a required property at 
$ echo $?
1
```

### Example 3: Wrong Type

```bash
$ cat wrong-type.json
{
  "company": "ACME Corp",
  "eps": "two dollars fifty",  # should be number
  "revenue_millions": 1500
}

$ python3 scripts/validate.py wrong-type.json schema.json
INVALID: 'two dollars fifty' is not of type 'number' at /eps
$ echo $?
1
```

### Example 4: Using YAML Schema

```bash
$ cat validation-rules.yaml
type: object
required:
  - symbol
  - price
properties:
  symbol:
    type: string
  price:
    type: number
    minimum: 0

$ cat stock-data.json
{
  "symbol": "AAPL",
  "price": 150.25
}

$ python3 scripts/validate.py stock-data.json validation-rules.yaml
OK
```

## เกี่ยวข้องกับอะไรอีกบ้าง

- **deploy-managed-agent.sh**: เมื่อ subagent มี `output_schema` ใน manifest (บรรทัด 154)
  ```bash
  json=$(jq --argjson c "$sub_ids" '.callable_agents=$c | del(.output_schema)' <<<"$json")
  ```
  deployment harness ลบ output_schema ออกจาก agent manifest แต่ใช้มัน validate ผลลัพธ์

- **test-cookbooks.sh**: ตรวจสอบว่า --dry-run resolve bodies ไม่มี `output_schema` leaked เข้าไป

- **Harness integration**: ในระบบจริง output_schema ใน subagent manifest อยู่เพื่อ:
  1. ให้ harness รู้ว่าต้องเรียก `validate.py` กับ output
  2. agent ทราบว่า output ต้อง conform กับ schema นี้
  3. ระหว่าง subagent → orchestrator มีการ validate

## Common Schemas สำหรับ Financial Agents

```yaml
# earnings_extractor subagent
earnings_schema:
  type: object
  required: [eps, revenue, net_income]
  properties:
    eps: {type: number}
    revenue: {type: number}
    net_income: {type: number}
    currency: {type: string, default: "USD"}

# market_researcher subagent
market_schema:
  type: object
  required: [symbols, market_cap, sector]
  properties:
    symbols: {type: array, items: {type: string}}
    market_cap: {type: number}
    sector: {type: string}

# gl_reconciler subagent
gl_schema:
  type: object
  required: [gl_accounts, total_debits, total_credits]
  properties:
    gl_accounts: {type: array, minItems: 1}
    total_debits: {type: number, minimum: 0}
    total_credits: {type: number, minimum: 0}
```
