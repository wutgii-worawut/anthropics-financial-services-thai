---
title: Earnings Reviewer
parent: 06 Managed Agent Cookbooks
nav_order: 2
---

# Earnings Reviewer Cookbook

## โครงสร้าง Agent

**Earnings Reviewer** เป็น managed agent ที่อ่านและวิเคราะห์ earnings call transcripts จากนั้นอัปเดตโมเดลทางการเงินโดยอัตโนมัติ โดยใช้ API inputs

```yaml
# จาก agent.yaml (บรรทัด 3-18)
name: earnings-reviewer
model: claude-opus-4-7

system:
  file: ../../plugins/agent-plugins/earnings-reviewer/agents/earnings-reviewer.md
  append: "You are running headless. Produce files in ./out/; do not assume an open Office document."

tools:
  - type: agent_toolset_20260401
    default_config: { enabled: false }
    configs:
      - { name: read,  enabled: true }
      - { name: grep,  enabled: true }
      - { name: glob,  enabled: true }
  - { type: mcp_toolset, mcp_server_name: factset, default_config: { enabled: true } }
  - { type: mcp_toolset, mcp_server_name: daloopa, default_config: { enabled: true } }
```

## Subagents แบ่งความเป็นส่วนตัว (Security Tiers)

Earnings Reviewer ประกอบด้วย 3 subagents ที่มีการแบ่งเขตความปลอดภัยเพราะ **transcripts เป็น untrusted input**:

| Tier | Subagent | Touches Untrusted Docs? | Tools | Connectors | บทบาท |
|------|----------|---|-------|---|---|
| 1 | `transcript-reader` | **YES** ⚠️ | read, grep only | None | อ่าน transcripts ที่อาจเป็น untrusted |
| 2 | `model-updater` | NO | read, grep, glob, bash (sandboxed) | FactSet, Daloopa (read-only) | Update Excel model |
| 3 | `note-writer` | NO | read, write, edit | None | เขียนเอกสาร supporting |

### Subagent 1: Transcript Reader (Untrusted Input Sanitizer)

**บทบาท**: ตรวจสอบ earnings call transcript และแยกข้อมูลสำคัญ

**Operations**:
- อ่าน transcript file (`.txt`, `.pdf`, `.docx`)
- Extract:
  - Management guidance (revenue, EPS, margins)
  - Key business themes (e.g., "market weakness", "product momentum")
  - Tone indicators (bullish, cautious, neutral)
  - Analyst Q&A themes (what analysts pressed on)
- **Length-cap all outputs** เพื่อป้องกัน prompt injection (e.g., max 10,000 tokens of extracted text)
- **Return schema-validated JSON only** — ไม่เคยส่ง free-form text

**Output Schema (Strict Validation)**:
```json
{
  "ticker": "AAPL",
  "call_date": "2026-05-10",
  "fiscal_period": "Q2 FY2026",
  "guidance": {
    "fy2026_revenue": { "low": "95B", "high": "98B" },
    "fy2026_eps": { "low": "7.50", "high": "7.75" },
    "margin_commentary": "Expect gross margin expansion from Services growth"
  },
  "key_themes": [
    { "theme": "services growth", "evidence": "Services revenue +18% YoY" },
    { "theme": "geographic weakness", "evidence": "Greater China slightly down YoY" }
  ],
  "tone": "cautious_optimism",
  "qa_themes": [
    "iPhone demand trajectory",
    "AI capabilities timeline",
    "Services pricing power"
  ],
  "data_quality": {
    "transcript_completeness": 95,
    "extracted_claims": 23,
    "capped_sections": 0
  }
}
```

### Subagent 2: Model Updater

**บทบาท**: ใช้ guidance จาก transcript-reader เพื่ออัปเดต Excel model

**Operations**:
- รับ structured JSON จาก transcript-reader
- รับ current Excel model file (`.xlsx`)
- ใช้ FactSet data เพื่อ:
  - Verify guidance เทียบกับ consensus estimates
  - Pull updated historical actuals
  - Compute implied changes to valuation
- Modify Excel cells:
  - Update FY guidance cells (e.g., revenue cell → new guidance range)
  - Update assumptions (capex %, R&D %)
  - Re-calculate DCF valuation
  - Highlight changed cells
- **Return updated model + change log**

**Change Log Format**:
```
FY2026 Revenue Guidance:
  Prior: $90B–$92B
  Updated: $95B–$98B
  Rationale: Management cited strong demand

Gross Margin Guidance:
  Prior: 42%–43%
  Updated: 43%–44% (Management commentary on mix shift)

DCF Valuation Impact:
  Prior: $155/share
  Updated: $162/share (+4.5%)
```

### Subagent 3: Note-Writer (Write Holder)

**บทบาท**: สร้างเอกสาร supporting document (analyst note)

**Operations**:
- รับ data จาก transcript-reader และ model-updater
- สร้างเอกสาร Word สำหรับ analyst note:
  - Summary (2 paragraphs)
  - Key takeaways (5 bullet points)
  - Model changes (table)
  - Revised view (1 paragraph with rationale)
  - Financial table (old vs. new estimates)
- **Only write permission holder** — สร้าง `./out/note-{ticker}.docx`

## Headless Operation Flow

```
API Request: { ticker, transcript_path, model_path, ... }
    ↓
Earnings Reviewer Orchestrator
    ├─→ transcript-reader
    │   Input: transcript file
    │   Output: JSON { guidance, themes, tone, qa_themes }
    │   ⚠️ UNTRUSTED INPUT HANDLING: max tokens, schema validation, length capping
    │
    ├─→ model-updater (with transcript-reader output)
    │   Input: JSON guidance + Excel model + FactSet data
    │   Output: updated Excel + change log
    │
    └─→ note-writer (with all prior outputs)
        Input: extracted data + model changes
        Output: analyst note DOCX
         └─→ Writes to ./out/
```

## MCP Servers ที่ต้องการ

### 1. FactSet MCP Server

```bash
export FACTSET_MCP_URL="https://your-factset-mcp-server"
```

Tools provided:
- `get_earnings_consensus()` — Consensus estimates (mean, median, range)
- `get_guidance_history()` — Prior period guidance เทียบกับ actual
- `get_analyst_recommendations()` — Rating distribution
- `get_financial_actuals()` — Historical quarterly/annual data

### 2. Daloopa MCP Server

```bash
export DALOOPA_MCP_URL="https://your-daloopa-mcp-server"
```

Tools provided:
- `get_market_reaction()` — Stock move post-earnings
- `get_sentiment_data()` — Real-time sentiment from news
- `get_estimate_revisions()` — Est. revisions last 30 days

## การ Deploy

### 1. Environment Setup

```bash
export ANTHROPIC_API_KEY=sk-ant-...
export FACTSET_MCP_URL="https://your-factset-server"
export DALOOPA_MCP_URL="https://your-daloopa-server"

# Validate connectivity
curl -s $FACTSET_MCP_URL/health
curl -s $DALOOPA_MCP_URL/health
```

### 2. Deploy Agent

```bash
../../scripts/deploy-managed-agent.sh earnings-reviewer
```

### 3. Invoke via API

```bash
curl -X POST https://api.claude.ai/v1/agents/earnings-reviewer/run \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "company_ticker": "AAPL",
    "transcript_path": "./transcripts/aapl_q2_2026_earnings.txt",
    "current_model_path": "./models/aapl_financial_model_20260501.xlsx",
    "analyst_name": "Apple Research Team",
    "update_type": "full"
  }'
```

## โครงสร้าง Output

```json
{
  "status": "completed",
  "company": "Apple Inc",
  "ticker": "AAPL",
  "call_date": "2026-05-10",
  "earnings_date": "2026-05-10",
  "fiscal_period": "Q2 FY2026",
  "execution_time_ms": 38000,
  
  "transcript_analysis": {
    "guidance_fy2026": {
      "revenue": "95B-98B",
      "gross_margin": "43%-44%",
      "eps": "7.50-7.75",
      "guidance_change": "raised"
    },
    "key_themes": [
      "services growth acceleration",
      "geographic weakness in Greater China",
      "capex investment in AI infrastructure"
    ],
    "tone": "cautious_optimism",
    "confidence_level": "high"
  },
  
  "model_updates": {
    "file": "aapl_model_updated_20260512.xlsx",
    "changes_made": [
      {
        "cell": "FY2026_Revenue_Guidance",
        "prior": "90.0B",
        "updated": "96.5B",
        "magnitude": "+7.2%",
        "source": "Management guidance"
      },
      {
        "cell": "Gross_Margin_FY2026",
        "prior": "0.4200",
        "updated": "0.4350",
        "magnitude": "+150bps",
        "source": "Management commentary + mix analysis"
      }
    ],
    "valuation_impact": {
      "prior_dcf_per_share": 155.20,
      "updated_dcf_per_share": 162.45,
      "change_pct": "+4.6%",
      "key_driver": "Revenue upside + margin expansion"
    }
  },
  
  "analyst_note": {
    "file": "aapl_earnings_note_20260512.docx",
    "pages": 4,
    "sections": [
      "executive_summary",
      "key_takeaways",
      "transcript_highlights",
      "model_changes",
      "revised_view",
      "financial_summary_table",
      "risk_factors"
    ]
  },
  
  "quality_metrics": {
    "transcript_completeness": 98,
    "extraction_confidence": 0.92,
    "model_update_validation": "passed",
    "data_source_confirmation": "FactSet and Daloopa cross-checked"
  }
}
```

## Data Parsing Strategy — Earnings Transcripts

### Approach 1: Structured Transcript Files

Jika transcript sudah dalam format `.json` atau `.xlsx` dengan clear sections (prepared remarks, Q&A):

```
{
  "call_metadata": {
    "ticker": "AAPL",
    "call_date": "2026-05-10",
    "fiscal_period": "Q2 FY2026"
  },
  "prepared_remarks": {
    "speakers": [...],
    "text": "..."
  },
  "qa_section": {
    "analyst_name": "Jane Smith (Goldman Sachs)",
    "question": "Can you elaborate on Services growth?",
    "answer": "..."
  }
}
```

**Action**: Transcript-reader uses grep/read tools langsung ke structured fields.

### Approach 2: Unstructured Text Transcripts

Jika transcript dalam plain `.txt` atau `.docx`:

```
APPLE INC. - Q2 FY2026 EARNINGS CALL
May 10, 2026

[Opening Remarks by CEO Tim Cook]
"Good afternoon, and thank you for joining us. ...
[dense prose, 50+ pages]
"

[Q&A Session]
"Goldman Sachs Analyst: Question about Services?"
"Tim Cook: Thanks for the question..."
```

**Action**: Transcript-reader:
1. Uses `grep` to locate patterns ("guidance", "outlook", "margin", "Services")
2. Extracts 2-3 sentence context around each match
3. Assembles into structured JSON (guidance, themes)
4. Validates extraction (cross-check: does "revenue guidance" appear 3+ times? Confidence is higher)

### Approach 3: PDF Transcripts

```bash
# Extract text first
pdftotext aapl_q2_earnings.pdf aapl_q2_earnings.txt

# Then process as unstructured text
```

## Model Updates — Excel Automation

### Architecture

```
Excel Model Structure:
├── Input Sheet
│   ├── FY2026 Revenue Guidance (cell A1: "95B-98B")
│   ├── Gross Margin Assumption (cell A2: "43%-44%")
│   └── Other inputs
│
├── Financials Sheet
│   ├── Revenue row (formulas reference Input sheet)
│   ├── Gross Profit (= Revenue × Margin)
│   └── EPS row
│
├── DCF Sheet
│   ├── Assumptions (WACC, terminal growth)
│   ├── Projection (5-year forecast)
│   └── Valuation
│
└── Sensitivity Sheet
    ├── WACC sensitivity table
    └── Revenue growth sensitivity
```

### Update Process (model-updater)

```javascript
// Pseudocode
const workbook = readXLSX(modelPath);

// 1. Update guidance cells
workbook.getSheet("Input").setCell("A1", "95.0B");  // min
workbook.getSheet("Input").setCell("B1", "98.0B");  // max

// 2. Recalculate all dependent cells
workbook.recalculate();

// 3. Compare results
const priorValuation = 155.20;
const updatedValuation = workbook.getSheet("DCF").getCell("Valuation").value;

// 4. Document changes
const changelog = [
  { field: "Revenue Guidance", old: "90B-92B", new: "95B-98B" },
  { field: "DCF Value/Share", old: "155.20", new: updatedValuation }
];

// 5. Write updated model + changelog
writeXLSX(workbook, "./out/aapl_model_updated.xlsx");
writeJSON(changelog, "./out/aapl_changes.json");
```

## Accuracy Considerations

### 1. Guidance Extraction

**ความอันตราย**: Management อาจให้ guidance เป็น ranges, conditionals, หรือ qualitative statements

```
Management statement: "We expect revenue in the range of $95B to $98B, 
assuming no material changes in FX or macro environment"
```

**Solution**:
- Always extract ปลาย high/low ของ range
- Note conditions ("assuming...")
- Flag guidance as "conditional" if caveats present

### 2. Consensus vs. Guidance Mismatch

**ความอันตราย**: FactSet consensus อาจตรงกันข้ามกับ management guidance

```
Management: "revenue $95B–$98B"
FactSet consensus: mean $90.5B (because call was surprising)
```

**Solution**:
- Compare guidance vs. prior consensus ใน FactSet
- Compute "beat/miss" on a per-metric basis
- Flag large mismatches (> 5%)
- Document which one is used ในโมเดล

### 3. Prior Model Integrity

**ความอันตราย**: Existing Excel model อาจมี hardcoded values or broken formulas

**Solution** (model-updater):
- Check all formula cells for circular references: `workbook.validate()`
- If validation fails, report error to user (don't overwrite)
- Update input cells only, never formulas
- Create backup: `cp ./out/aapl_model.xlsx ./out/aapl_model_backup.xlsx`

## Edge Cases

### Case 1: Negative EPS Guidance

**Situation**: Company guides to loss for quarter (e.g., "we expect EPS loss of $(0.15) to $(0.10)")

**Handling**:
- transcript-reader extracts: `{ "eps": { "low": -0.15, "high": -0.10 } }`
- model-updater updates cells with negative values
- DCF model may treat this as temporary → no change to terminal value
- Analyst note flags: "Company guiding to near-term EPS loss due to [reason]"

### Case 2: No Guidance Provided

**Situation**: Management explicitly withdraws guidance ("We are not providing FY2026 guidance at this time")

**Handling**:
- transcript-reader extracts: `{ "guidance": null, "note": "guidance withdrawn" }`
- model-updater skips guidance cells, uses prior assumptions
- note-writer states: "Management maintains prior guidance pending [condition]"

### Case 3: Guidance Change Direction Reversal

**Situation**: Management raised guidance 3 quarters ago, now lowers

```
Prior call (3 months ago): FY2026 revenue $98B–$102B
Current call: FY2026 revenue $95B–$98B (lowered)
```

**Handling**:
- FactSet has `get_guidance_history()` — pull prior guidance automatically
- model-updater computes delta: "Guidance revised down $3B–$4B"
- note-writer highlights: "First guidance reduction after 3 quarters of raises"

## ความแตกต่างจาก Interactive Plugin

**Earnings Reviewer Plugin** (Interactive):
- ผู้ใช้อัปโหลด transcript
- Agent ถามคำถาม clarifying ("Did they guide to 30% margin or 30bps change?")
- ผู้ใช้เห็นผลลัพธ์ step-by-step
- ใช้สำหรับการวิเคราะห์ ad-hoc, conversation-driven

**Earnings Reviewer Cookbook** (Managed):
- Fully automated, API-triggered
- ไม่มี user interaction
- Output files ไปที่ `./out/` โดยอัตโนมัติ
- ใช้สำหรับ batch processing (ตัวอย่าง: automate model updates for 50-stock coverage list)

## Steering Events & Orchestration

### Example Steering Events (จาก steering-examples.json)

```json
[
  {
    "event": "Process earnings: NVDA Q1-FY27",
    "description": "Single ticker, single period — one-off analyst request"
  },
  {
    "event": "Process earnings: coverage-list semis, period Q1-FY27",
    "description": "Fan-out across coverage list — orchestration layer iterates once per ticker"
  },
  {
    "event": "Update model only: NVDA Q1-FY27, skip note",
    "description": "Follow-up when analyst writes note themselves, only update model"
  }
]
```

### Orchestration Usage

```bash
# Single ticker
curl -X POST https://api.claude.ai/v1/agents/earnings-reviewer/run \
  -d '{ "event": "Process earnings: AAPL Q2-FY26", "ticker": "AAPL" }'

# Batch across coverage list (orchestrator loops)
for ticker in AAPL MSFT GOOGL NVDA AMD INTC; do
  curl -X POST https://api.claude.ai/v1/agents/earnings-reviewer/run \
    -d "{ \"event\": \"Process earnings: $ticker Q2-FY26\" }" &
done
wait
```

---

**หมายเหตุ**: Model update ใช้ Excel API — โมเดลต้องเป็น structured spreadsheet ที่มี defined ranges สำหรับ input assumptions และ output formulas
