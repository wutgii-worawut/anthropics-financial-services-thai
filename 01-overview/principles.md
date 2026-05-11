---
title: หลักการออกแบบ
parent: 01 ภาพรวม
nav_order: 5
---

# หลักการออกแบบ 5 ประการ

Claude for Financial Services มีหลักการ 5 ข้อ ที่กำหนดการออกแบบ architecture นั้น ไม่ใช่ "best practices" ทั่วไป — หลักการเหล่านี้เกิดจากบทเรียนตรงมาจากการทำงานจริงใน FSI

## 1. Vertical-First: Single Source of Truth per Domain

**หลักการ**: Skill ต้องมี source แหล่งเดียวใน vertical plugin; agent bundles เป็นแค่ copies ที่ sync

### เหตุผล

**Problem**: สถาบันการเงิน 10 แห่ง ต้องการ DCF model ที่ต่างกันเล็กน้อย (ค่า WACC, market cap definition, dividend policy)
- ❌ Copy DCF skill ไปยัง 10 agents → 10 ตัวออกจากซิงค์ → inconsistency nightmare
- ✅ 1 DCF skill ใน `financial-analysis` → ทั้ง 10 agents ดึง copy ผ่าน `sync-agent-skills.py`

(README.md line 89–99, CLAUDE.md line 31)

### วิธีการ

```bash
# EDIT SKILL ONCE:
vim plugins/vertical-plugins/financial-analysis/skills/dcf-model/SKILL.md
# ↓ Change: Terminal WACC assumptions, margin build logic, sensitivity table convention

# PROPAGATE AUTOMATICALLY:
python3 scripts/sync-agent-skills.py

# VERIFY NO DRIFT:
python3 scripts/check.py
# ✓ All bundled skills match vertical sources
# ✓ No skill is stale
```

(CLAUDE.md line 31–32)

### ได้ประโยชน์

- **Maintenance**: Edit skill ชิ้นเดียว = ได้ผล 4 agents
- **Consistency**: Pitch Agent + Model Builder Agent = ใช้ DCF logic เดียวกัน ไม่ได้ implement ต่างแบบคนละแบบ
- **Discoverability**: Knowledge base agent นอกจาก vertical = "ที่ไหนมี DCF skill?" → `plugins/vertical-plugins/financial-analysis/skills/dcf-model/`
- **Reuse**: ลองสร้าง agent ใหม่? Copy structure เลือก existing skills ใน verticals
- **Vertical growth**: PE vertical เพิ่ม `/ai-readiness` skill → automatic ใหม่ทั้ง private-equity agents ที่ bundle มัน

---

## 2. Manifest-Driven Quality: No Unit Tests, Lint + Drift Detection

**หลักการ**: Skill quality มาจากการ lint metadata + drift detection, ไม่ใช่ unit tests

### เหตุผล

**Problem**: 
- Unit tests เขียนยาก สำหรับ agent workflows (LLM non-deterministic, mocks heavy, slow)
- Agent skills โปรแกรมแบบ prose (markdown SKILL.md) + prompts (system message) — ไม่ testable เหมือนโค้ด

**Solution**: Manifest-driven validation
- ✅ `check.py` lint every plugin.json + agent reference
- ✅ Verify all `{file:...}` paths exist + resolve
- ✅ Verify skill bundles haven't drifted from verticals
- ✅ Verify MCP configurations are valid
- ✅ Block commit ถ้า something wrong

(CLAUDE.md line 31)

### ตัวอย่าง

```bash
# publish ก่อน:
python3 scripts/check.py

# Output ถ้ามีปัญหา:
# ERROR: plugins/agent-plugins/pitch-agent/skills/dcf-model/SKILL.md
#        differs from plugins/vertical-plugins/financial-analysis/skills/dcf-model/SKILL.md
#        (drift detected by sync-agent-skills.py)
# → Run sync-agent-skills.py first
```

### ได้ประโยชน์

- **Safety**: ไม่ได้ push code ที่มี broken references
- **Deterministic**: ทั้ง Cowork + Managed Agents ใช้ไฟล์เดียวกัน
- **Authoring**: People เขียน markdown SKILL.md, tools validate ว่าสมดุล
- **CI/CD**: `check.py` ไว้ใน pre-commit hook / GitHub Actions

---

## 3. Hard-Allowlist Orchestration: Don't Trust LLM-to-LLM Output

**หลักการ**: Agent ไม่เชื่อถือ output จาก agent อื่นโดยตรง — แทนที่ใช้ whitelist tools และ MCP servers

### เหตุผล

**Problem**: 
- Subagent A ส่ง output ไป Subagent B
- Subagent B ไม่มี context ว่า A ทำอะไร (hallucination risk)
- Subagent B trust A's output แล้ว execute action ที่ destructive (post to ledger, execute trade, commit data)

**Solution**: Hard allowlist
- ❌ Subagent A ไม่สามารถ call B directly
- ✅ Orchestrator (human หรือ script) ตรวจสอบ A's output → route ไป B พร้อม explicit instructions

(managed-agent-cookbooks/gl-reconciler/agent.yaml line 16–37)

### ตัวอย่าง: GL Reconciler

```yaml
# managed-agent-cookbooks/gl-reconciler/agent.yaml

# Orchestrator (read-only, no side effects)
tools:
  - type: agent_toolset_20260401
    configs:
      - name: read          # ✅ allowed
        enabled: true
      - name: grep          # ✅ allowed
        enabled: true
      - name: write         # ❌ NOT allowed
        enabled: false
      - name: execute       # ❌ NOT allowed
        enabled: false

# Callable subagents (depth-1 only)
callable_agents:
  - manifest: ./subagents/reader.yaml      # Read GL data, flag breaks
  - manifest: ./subagents/critic.yaml      # Analyze findings
  - manifest: ./subagents/resolver.yaml    # Trace root cause

# MCP access (explicit allowlist)
mcp_servers:
  - type: url
    name: internal-gl              # Read-only GL server
  - type: url
    name: subledger                # Read-only subledger
```

**Orchestration flow:**
1. Orchestrator (orchestrate.py) sends GL data to Reader
2. Reader analyzes, returns "found discrepancy in AR reconciliation"
3. Orchestrator (human oversight point) **reviews Reader output**
4. If approved, route to Critic with explicit context: "Check AR reconciliation" 
5. Critic hypothesizes root cause
6. Orchestrator reviews Critic output
7. Route to Resolver: "Update GL account XYZ per hypothesis H, then route for approval"
8. Resolver generates journal entry
9. Orchestrator sends to human sign-off (no agent posts to ledger)

(scripts/orchestrate.py, managed-agent-cookbooks/gl-reconciler/README.md)

### ได้ประโยชน์

- **Auditability**: ทุก handoff record ได้ → history trail
- **Human oversight**: Critical action (GL post, trade execution, approval) = human approve
- **Regulatory compliance**: สำหรับ fund operations (NAV posting, LP statements) = bank-grade controls
- **Risk containment**: Agent crash ไม่เกิด waterfall failure ในโปรแกรม cascade

---

## 4. Domain Validators: Business Rules Embedded in Code, Not Just Prompts

**หลักการ**: Rules ที่สำคัญ (DCF constraints, KYC rules, GL account structures) ต้องเป็น hard constraints ในไฟล์ skill, ไม่ใช่ prompt suggestion

### เหตุผล

**Problem**: 
- "Please build DCF with formulas, not hardcodes" ✍️ ← ในโปรแกรม hint ไม่ guarantee
- Skill ดึง data มา → compute ผิด → hardcoded cell แทน formula → model flexible

**Solution**: Embed rules ในไฟล์ SKILL.md เป็น step-by-step guide + constraints

(plugins/vertical-plugins/financial-analysis/skills/dcf-model/SKILL.md line 18–55)

### ตัวอย่าง: DCF Constraints

```markdown
---
name: dcf-model
description: Real DCF model creation...
---

# DCF Model Builder

## Critical Constraints - Read These First

**Formulas Over Hardcodes (NON-NEGOTIABLE):**
- Every projection, margin, discount factor, PV, sensitivity cell MUST be a live Excel formula
- ❌ WRONG: ws["D20"] = calculated_revenue  (hardcoded number)
- ✅ CORRECT: ws["D20"] = "=D19*(1+$B$8)"    (live formula)
- The only hardcoded numbers: raw historical inputs, assumption drivers, market data

**Sensitivity Tables:**
- Use odd dimensions (5×5, 7×7) — center cell = base case
- If WACC = 9.0%, middle row = 9.0%; if terminal g = 3.0%, middle column = 3.0%
- Center cell output = actual implied share price (sanity check)

**Verify Step-by-Step:**
- After data retrieval → show user raw inputs, confirm
- After revenue projections → show top line + growth rates, confirm
- After FCF build → show FCF schedule, confirm before WACC
- After WACC → show calculation, confirm before discounting
- After PV → show equity bridge, confirm before sensitivity
```

Skill เป็นไม่เพียง "guidance" — เป็น **specification** ที่ Agent follow step-by-step

### ได้ประโยชน์

- **Correctness**: DCF formulas ตามธรรมเนียมการลงทุนแบงก์ (blue = input, black = formula, green = output)
- **Auditability**: model สามารถ trace ได้ (ทั้ง formula และ assumption)
- **Compliance**: KYC rules (age 18+, AML check, accredited investor) embedded ใน operations/kyc-parser/SKILL.md → agent follow, ไม่ bypass
- **Quality**: ทรวจสอบสำหรับ novice analyst (รูป formula ไม่ hardcode ได้เรียนรู้ตั้งแต่ต้น)

### ตัวอย่าง: KYC Rules

```markdown
# KYC Parser

## Rules Engine (Embedded)

Every onboarding document MUST pass these checks:

1. **Accreditation**: Document must show income >= $200k or net worth >= $1M (§506(c) rule)
   - ❌ FAIL: "I have some money but no docs" → flag & pause
   - ✅ PASS: Tax return + bank statements → continue

2. **AML/Sanctions**: Run name against OFAC list
   - ❌ HIT: Name matches → file SAR, REJECT application
   - ✅ MISS: No match → continue

3. **Beneficial Owner**: Document beneficiary ownership > 20%
   - ❌ FAIL: "Trust but trustee not disclosed" → request trust agreement
   - ✅ PASS: Individual or disclosed trust → continue
```

ไม่ได้ rely on Claude "trying" — rules ลงรหัสด้วย decision tree

---

## 5. MCP Over Web Search: Allowlisted Data Sources, No Open Web

**หลักการ**: Agent ดึงข้อมูลจาก MCP servers (closed, allowlisted) ไม่ใช่ web search

### เหตุผล

**Problem**: Web search ใน FSI = risk
- Hallucination: Claude บอก "Apple stock was $180 in 2020" เท็จ
- Stale data: Search ได้ข้อมูลเก่า 6 เดือน → analyst ใช้ stale comps
- Compliance: ไม่รู้ข้อมูล มาจากไหน → audit trail สำคัญ
- No guarantee data source แน่นอน

(README.md line 119–135, plugins/vertical-plugins/financial-analysis/.mcp.json)

**Solution**: MCP servers (allowlisted, credentialed)
- ✅ Daloopa — live financial metrics
- ✅ CapIQ — M&A transactions, trailing multiples
- ✅ FactSet — alternative data, verified research
- ✅ LSEG — bond analytics, FX curves
- ✅ Moody's — credit ratings, guaranteed from publisher

(README.md line 123–135)

### ตัวอย่าง: Comps Analysis

```
/comps Apple

1. Agent ไม่ search web → "top 10 Apple competitors"
2. ❌ ไม่ดึง Wikipedia
3. ✅ Query CapIQ MCP: "Market cap cohort near $3T, sector Tech Hardware"
4. ✅ Get: Peers = Samsung, Intel, TSMC, Nvidia, Broadcom, Qualcomm, Marvell, Analog Devices, ...
5. ✅ Query Daloopa MCP: Get trailing multiples (EV/Revenue, EV/EBITDA, P/E)
6. ✅ Return comps table with source = CapIQ + Daloopa
7. ✅ Audit trail: "Used CapIQ at 2026-05-12 15:43 UTC for peer set; used Daloopa for multiples at..."
```

Agent ไม่เดา — แหล่งข้อมูลชัดเจนและ verifiable

### การเชื่อมต่อ MCP

```json
// plugins/vertical-plugins/financial-analysis/.mcp.json
{
  "mcpServers": {
    "daloopa": {
      "type": "http",
      "url": "https://mcp.daloopa.com/server/mcp"
    },
    "sp-global": {
      "type": "http",
      "url": "https://kfinance.kensho.com/integrations/mcp"
    },
    ...  // 11 providers total
  }
}
```

Managed Agent deployment ตั้ง MCP credentials ใน environment:

```bash
# .env file
DALOOPA_API_KEY=...
CAPIQ_API_KEY=...
FACTSET_API_KEY=...
...
```

(managed-agent-cookbooks/gl-reconciler/agent.yaml line 30–37)

### ได้ประโยชน์

- **Accuracy**: Data แหล่งจากผู้ให้ใบอนุญาต (CapIQ = S&P Global, Daloopa = Daloopa)
- **Auditability**: "Data pulled from CapIQ at 2026-05-12 15:43 UTC" → reproducible
- **Compliance**: "ใคร verify data?" → "CapIQ MCP server บน credentials ของ firm"
- **Fallback**: ถ้า MCP ลง → agent know ว่า "data unavailable" ไม่ hallucinates
- **Customization**: Custom vertical → custom MCP (internal GL server, internal CRM, ...)

---

## การทำงานร่วมของหลักการ 5 ข้อ

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  User Prompt: "Build DCF for Acme Corp"                    │
│                        ↓                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Principle 1: Vertical-First                         │   │
│  │ → Find dcf-model skill in financial-analysis        │   │
│  │ → Agent bundles it (already synced via check.py)    │   │
│  └─────────────────────────────────────────────────────┘   │
│                        ↓                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Principle 2: Manifest-Driven Quality                │   │
│  │ → SKILL.md has step-by-step + constraints           │   │
│  │ → "Formulas over hardcodes", "Verify step-by-step"  │   │
│  │ → If constraint violated → agent surfaces error     │   │
│  └─────────────────────────────────────────────────────┘   │
│                        ↓                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Principle 5: MCP Over Web                           │   │
│  │ → Agent queries CapIQ MCP for Acme filings          │   │
│  │ → Agent queries Daloopa MCP for historical revenue  │   │
│  │ → No web search; data source = MCP (auditable)      │   │
│  └─────────────────────────────────────────────────────┘   │
│                        ↓                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Principle 4: Domain Validators                      │   │
│  │ → Agent follow SKILL.md step-by-step                │   │
│  │ → Step 1: Verify inputs (confirm growth rate %)     │   │
│  │ → Step 2: Build revenue projection with formulas    │   │
│  │ → ... (each step has validation rules)              │   │
│  │ → Step 8: Generate sensitivity table (5×5 odd)      │   │
│  │ → Center cell = base case (sanity check)            │   │
│  └─────────────────────────────────────────────────────┘   │
│                        ↓                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Principle 3: Hard-Allowlist Orchestration           │   │
│  │ → Managed Agent deployment (agent.yaml)             │   │
│  │ → Agent can READ Excel + MCP, CANNOT WRITE to ledger│   │
│  │ → Surface model for human approval                  │   │
│  │ → If approved, route to next stage (post to ledger) │   │
│  │   via explicit choreography (orchestrate.py)        │   │
│  └─────────────────────────────────────────────────────┘   │
│                        ↓                                    │
│  Output: DCF Model (live formulas, audit trail)            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## ดูเพิ่มเติม

- [What is it?](/01-overview/what-is-it.md) — 10 agents, 7 verticals, 2 runtimes
- [Architecture](/01-overview/architecture.md) — 3-layer reference: verticals → agents → cookbooks
- [Quick Start](/01-overview/quickstart.md) — ติดตั้งและลองใช้
