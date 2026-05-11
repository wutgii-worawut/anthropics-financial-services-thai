---
title: Valuation Reviewer
parent: 06 Managed Agent Cookbooks
nav_order: 3
---

# Valuation Reviewer Cookbook

## โครงสร้าง Agent

**Valuation Reviewer** เป็น managed agent ที่ตรวจสอบและวิเคราะห์ valuation models เพื่อให้แน่ใจว่า:
1. Models มี economic sense (assumptions ที่สมเหตุสมผล)
2. Formulas ถูกต้องทางคณิตศาสตร์
3. Output valuations สอดคล้องกับ peer benchmarks

```yaml
# จาก agent.yaml (บรรทัด 3-17)
name: valuation-reviewer
model: claude-opus-4-7

system:
  file: ../../plugins/agent-plugins/valuation-reviewer/agents/valuation-reviewer.md
  append: "You are running headless. Produce files in ./out/; do not assume an open Office document."

tools:
  - type: agent_toolset_20260401
    default_config: { enabled: false }
    configs:
      - { name: read,  enabled: true }
      - { name: grep,  enabled: true }
      - { name: glob,  enabled: true }
  - { type: mcp_toolset, mcp_server_name: portfolio, default_config: { enabled: true } }
```

## Subagents แบ่งบทบาท

Valuation Reviewer ประกอบด้วย 3 subagents ที่มีการแบ่งเขตความเป็นส่วนตัว (security tiers) เพราะ **GP-provided valuation packages อาจเป็น untrusted input**:

| Tier | Subagent | Touches Untrusted Docs? | Tools | MCP Connectors | บทบาท |
|------|----------|---|-------|---|---|
| 1 | `package-reader` | **YES** ⚠️ | read, grep only | None | อ่าน/parse valuation package |
| 2 | `valuation-runner` | NO | read, grep, glob, bash (sandboxed) | portfolio (read-only) | Run sensitivity analysis, validation |
| 3 | `publisher` | NO | read, write, edit | None | สร้างเอกสาร reporting |

### Subagent 1: Package Reader (Untrusted Input Sanitizer)

**บทบาท**: อ่านและ parse valuation package (DCF, comps, precedents) จากไฟล์ที่อาจเป็น untrusted

**Operations**:
- อ่าน Excel/CSV files ที่มี valuation data
- Extract key information:
  - DCF assumptions (WACC, terminal growth rate, capex intensity, NWC %)
  - Historical financials (revenue, EBITDA, FCF 3-5 years)
  - Forward projections (if provided)
  - Trading comps (company, revenue multiple, EBITDA multiple)
  - Precedent transactions (deal value, purchase price multiples)
- **Validate structure**: Check ว่า files ต่างๆ มี required columns, format consistency
- **Return schema-validated JSON only**

**Output Schema**:
```json
{
  "company": "Portfolio Company Inc.",
  "package_id": "LPR-2026-Q1",
  "validation_status": "passed_structure",
  
  "dcf_section": {
    "assumptions": {
      "wacc": 0.085,
      "terminal_growth_rate": 0.025,
      "capex_pct_revenue": 0.03,
      "nwc_pct_revenue": 0.08,
      "tax_rate": 0.21
    },
    "historical_financials": [
      { "year": 2023, "revenue": 45000000, "ebitda": 9000000, "fcf": 6300000 },
      { "year": 2024, "revenue": 54000000, "ebitda": 11340000, "fcf": 7938000 }
    ],
    "formula_completeness": "complete",
    "data_integrity": "passed"
  },
  
  "trading_comps": {
    "count": 6,
    "companies": [
      { "name": "CompCo 1", "revenue_multiple": 4.5, "ebitda_multiple": 12.3 },
      { "name": "CompCo 2", "revenue_multiple": 5.2, "ebitda_multiple": 13.8 }
    ],
    "median_multiple": { "revenue": 4.8, "ebitda": 12.8 }
  },
  
  "precedent_transactions": {
    "count": 4,
    "transactions": [
      { "target": "Acq 1", "deal_value_usd_m": 250, "revenue_multiple": 3.8, "ebitda_multiple": 10.5 }
    ],
    "median_multiple": { "revenue": 3.5, "ebitda": 9.8 }
  },
  
  "quality_metrics": {
    "data_completeness": 0.95,
    "assumed_issues": []
  }
}
```

### Subagent 2: Valuation Runner (Calculation & Sensitivity Engine)

**บทบาท**: ใช้ validated data จาก package-reader เพื่อทำ validation checks และ sensitivity analysis

**Operations**:
- **Formula Integrity Check**: 
  - Verify ทั้งหมด DCF formulas: `PV(free cash flows) + Terminal Value = Enterprise Value`
  - Check comps multiples: ใช้ consistent revenue/EBITDA definitions
  - Cross-check: Implied valuations from comps vs. DCF base case (should be within 10-15%)

- **Assumptions Reasonableness Check**:
  - WACC vs. peer average (flag if >100bps off)
  - Terminal growth rate vs. long-term GDP growth (should be <3% in developed markets)
  - EBITDA margin vs. peer median (flag if outlier)
  - Capex intensity vs. industry norms (use Portfolio Analytics MCP if available)

- **Sensitivity Analysis**:
  - Single-variable sensitivity: WACC ±1%, terminal growth ±0.5%, revenue growth ±2%
  - Two-variable sensitivity: WACC vs. terminal growth (create 5x5 table)
  - Output: Valuation ranges and key drivers

- **Scenario Testing**:
  - Base case, upside case (higher growth, margin expansion), downside case
  - Compare: Implied valuation per share across scenarios

**Output**:
```json
{
  "dcf_validation": {
    "status": "passed",
    "checks": [
      { "formula": "PV(FCF) + TV", "status": "correct", "components": "15 year forecast period" },
      { "formula": "Terminal Value = Year5 FCF * (1 + g) / (WACC - g)", "status": "correct" }
    ]
  },
  
  "assumptions_reasonableness": {
    "wacc": {
      "value": 0.085,
      "peer_median": 0.0825,
      "delta_bps": 25,
      "flag": "low_moderate" 
    },
    "terminal_growth": {
      "value": 0.025,
      "range_ok": "2.0% - 2.5%",
      "flag": "normal"
    }
  },
  
  "sensitivity_analysis": {
    "base_case_value_per_share": 42.50,
    "wacc_sensitivity": {
      "range": "7.5% - 9.5%",
      "valuation_range": { "low": 38.00, "high": 48.25 }
    },
    "terminal_growth_sensitivity": {
      "range": "2.0% - 3.0%",
      "valuation_range": { "low": 37.50, "high": 50.00 }
    },
    "key_driver_ranking": [
      { "assumption": "WACC", "impact_magnitude": "high", "sensitivity_pct": 18 },
      { "assumption": "Terminal Growth", "impact_magnitude": "high", "sensitivity_pct": 22 },
      { "assumption": "Revenue Growth Year 1-5", "impact_magnitude": "medium", "sensitivity_pct": 15 }
    ]
  },
  
  "scenario_analysis": {
    "downside": { "assumptions": "conservative", "value_per_share": 35.25 },
    "base": { "assumptions": "midpoint", "value_per_share": 42.50 },
    "upside": { "assumptions": "optimistic", "value_per_share": 52.00 }
  }
}
```

### Subagent 3: Publisher (Output Document Creator)

**บทบาท**: สร้างเอกสาร reporting สำหรับ LP/stakeholders

**Operations**:
- รับ validation + sensitivity results จาก valuation-runner
- สร้าง Excel output file ที่มี:
  - Input assumptions sheet (copy from original package)
  - Validation findings (with pass/flag indicators)
  - Sensitivity tables (WACC, terminal growth, revenue growth)
  - Scenario comparison
  - Summary sheet with valuation range and key metrics
- สร้าง Word document ที่มี:
  - Executive summary (2-3 paragraphs)
  - Validation findings (pass/fail status per check)
  - Key sensitivities (which assumptions move value most)
  - Recommendations (if adjustments suggested)

**Write-holder only** — ไม่มี access ไปที่ Portfolio MCP

## Headless Operation Flow

```
API Request: { package_path, company_name, review_type, ... }
    ↓
Valuation Reviewer Orchestrator
    ├─→ package-reader
    │   Input: valuation package (Excel/CSV files)
    │   Output: JSON { dcf_section, comps, precedents, quality_metrics }
    │   ⚠️ UNTRUSTED INPUT: structure validation, data extraction only
    │
    ├─→ valuation-runner (with package-reader output)
    │   Input: JSON + Portfolio Analytics data
    │   Output: { validation_status, sensitivity_analysis, scenario_analysis }
    │   ✓ TRUSTED: all validation & calculation logic
    │
    └─→ publisher (with all prior outputs)
        Input: validation + sensitivity results
        Output: Excel + Word documents
         └─→ Writes to ./out/
```

## การ Deploy

### 1. Environment Setup

```bash
export ANTHROPIC_API_KEY=sk-ant-...
export PORTFOLIO_MCP_URL="https://your-portfolio-analytics-server"

# Verify connectivity
curl -s $PORTFOLIO_MCP_URL/health
```

### 2. Deploy Agent

```bash
../../scripts/deploy-managed-agent.sh valuation-reviewer
```

### 3. Invoke via API

```bash
curl -X POST https://api.claude.ai/v1/agents/valuation-reviewer/run \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "package_path": "./valuations/acme_corp_dcf.xlsx",
    "company_name": "Acme Corp",
    "review_type": "comprehensive",
    "sensitivity_factors": ["wacc", "terminal_growth", "capex_intensity", "revenue_growth"],
    "benchmark_against_peers": true
  }'
```

## โครงสร้าง Output

```json
{
  "status": "completed",
  "company": "Acme Corp",
  "execution_time_ms": 42000,
  
  "valuation_summary": {
    "base_case_equity_value_usd_b": 8.5,
    "equity_value_per_share": 42.50,
    "valuation_range": {
      "low_per_share": 38.00,
      "high_per_share": 48.00,
      "low_usd_b": 7.6,
      "high_usd_b": 9.6
    },
    "implied_multiples": {
      "ev_ebitda_2024": 12.3,
      "price_sales_2024": 2.8,
      "price_fcf": 14.5
    }
  },
  
  "review_findings": {
    "formula_integrity": {
      "status": "passed",
      "details": "All DCF formulas verified, PV(FCF) + TV calculations correct"
    },
    "assumptions_reasonableness": {
      "status": "passed_with_notes",
      "details": [
        "WACC 8.5% is 25bps above peer median 8.25% (acceptable)",
        "Terminal growth 2.5% is reasonable vs. long-term GDP",
        "Revenue CAGR 8% is conservative vs. comp median 10%"
      ]
    },
    "output_sensitivity": "medium",
    "valuation_dispersion": "WACC ±1% drives 18% valuation swing; Terminal growth ±0.5% drives 22% swing"
  },
  
  "sensitivity_analysis": {
    "wacc_sensitivity": {
      "range_tested": "7.5% - 9.5%",
      "valuation_impact": {
        "low_wacc": 48.25,
        "base_wacc": 42.50,
        "high_wacc": 38.00
      }
    },
    "terminal_growth_sensitivity": {
      "range_tested": "2.0% - 3.0%",
      "valuation_impact": {
        "low_growth": 37.50,
        "base_growth": 42.50,
        "high_growth": 50.00
      }
    },
    "revenue_growth_sensitivity": {
      "range_tested": "5% - 10%",
      "valuation_impact": {
        "low_growth": 39.00,
        "base_growth": 42.50,
        "high_growth": 47.50
      }
    }
  },
  
  "scenario_analysis": {
    "downside_scenario": {
      "assumptions": {
        "revenue_growth_cagr": "5%",
        "ebitda_margin": "18%",
        "wacc": "9.0%"
      },
      "equity_value_per_share": 35.25,
      "rationale": "Conservative case with macro headwinds"
    },
    "base_case": {
      "assumptions": {
        "revenue_growth_cagr": "8%",
        "ebitda_margin": "20%",
        "wacc": "8.5%"
      },
      "equity_value_per_share": 42.50,
      "rationale": "Midpoint of consensus expectations"
    },
    "upside_scenario": {
      "assumptions": {
        "revenue_growth_cagr": "12%",
        "ebitda_margin": "22%",
        "wacc": "7.5%"
      },
      "equity_value_per_share": 52.00,
      "rationale": "Optimistic case with operational improvements"
    }
  },
  
  "recommendations": [
    {
      "category": "assumption_adjustment",
      "finding": "Terminal growth rate at 2.5% appears elevated given conservative long-term GDP outlook",
      "recommendation": "Consider reducing to 2.0% for additional conservatism",
      "impact": "Would reduce per-share value to $39.75"
    },
    {
      "category": "valuation_verification",
      "finding": "Implied EV/EBITDA of 12.3x is 2.0x peer median of 6.1x",
      "recommendation": "Verify assumptions drive margin expansion to justify premium",
      "impact": "Critical to thesis credibility"
    }
  ],
  
  "output_files": {
    "validation_report": {
      "filename": "acme_valuation_review_report.docx",
      "pages": 6,
      "sections": ["Executive Summary", "Validation Findings", "Sensitivity Analysis", "Recommendations"]
    },
    "updated_package": {
      "filename": "acme_dcf_validated_20260512.xlsx",
      "sheets": ["DCF Model", "Comps", "Precedents", "Validation Results", "Sensitivity Tables", "Summary"]
    },
    "sensitivity_charts": {
      "filename": "acme_sensitivity_analysis_20260512.pdf",
      "charts": ["WACC Sensitivity", "Terminal Growth Sensitivity", "2-Way Sensitivity WACC vs. Growth", "Scenario Waterfall"]
    }
  },
  
  "metadata": {
    "review_completeness": "comprehensive",
    "sensitivities_tested": 4,
    "scenarios_analyzed": 3,
    "validation_checks_passed": 12,
    "validation_checks_flagged": 2,
    "data_quality_score": 0.92
  }
}
```

## Validation Rules Applied (Script Reference)

### 1. Formula Integrity Checks

```python
# Pseudocode from valuation-runner
def check_dcf_formula_integrity(dcf_model):
    """Verify DCF calculation structure"""
    
    # Check 1: Terminal Value formula
    tv_formula = dcf_model.get_cell("TerminalValue")
    # Expected: Year5_FCF * (1 + terminal_growth) / (WACC - terminal_growth)
    if not verify_formula_structure(tv_formula, expected_pattern):
        return ValidationError("Terminal Value formula incorrect")
    
    # Check 2: PV calculation
    pv_fcf = sum(dcf_model.get_range("DiscountedFCF"))
    pv_tv = dcf_model.get_cell("PresentValueTerminalValue")
    enterprise_value = pv_fcf + pv_tv
    
    # Cross-check: Enterprise Value should match total on summary sheet
    summary_ev = dcf_model.get_sheet("Summary").get_cell("EnterpriseValue")
    if abs(enterprise_value - summary_ev) > 1_000_000:  # Allow $1M rounding
        return ValidationError("Enterprise Value mismatch between sheets")
    
    return ValidationResult("passed", details="All formulas verified")
```

### 2. Assumptions Reasonableness Checks

```python
def check_assumptions_reasonableness(dcf_model, peer_data):
    """Compare assumptions to peer benchmarks"""
    
    issues = []
    
    # WACC check
    assumed_wacc = dcf_model.get_cell("WACC")
    peer_wacc_median = peer_data.get_metric("wacc_median")
    delta = abs(assumed_wacc - peer_wacc_median)
    if delta > 0.0100:  # >100bps difference
        issues.append({
            "assumption": "WACC",
            "assumed": assumed_wacc,
            "peer_median": peer_wacc_median,
            "flag": "moderate" if delta < 0.0150 else "high"
        })
    
    # Terminal growth check
    tg = dcf_model.get_cell("TerminalGrowthRate")
    if tg > 0.03:  # >3% in developed market
        issues.append({
            "assumption": "Terminal Growth",
            "value": tg,
            "recommendation": "Reduce to 2.0-2.5% for conservative developed market",
            "flag": "medium"
        })
    
    # EBITDA margin check
    proj_margin = dcf_model.get_range("EBITDA_Margin").mean()
    peer_margin = peer_data.get_metric("ebitda_margin_median")
    if proj_margin > peer_margin + 0.05:  # >500bps above peers
        issues.append({
            "assumption": "EBITDA Margin",
            "projected": proj_margin,
            "peer_median": peer_margin,
            "flag": "medium",
            "recommendation": "Verify margin expansion assumptions with operating plan"
        })
    
    return ValidationResult(
        status="passed_with_notes" if issues else "passed",
        issues=issues
    )
```

### 3. Sanity Checks

```python
def run_sanity_checks(dcf_model, company_data):
    """Basic smell tests"""
    
    checks = []
    
    # Revenue CAGR should be positive and reasonable
    rev_cagr = dcf_model.compute_cagr("Revenue")
    if rev_cagr < 0:
        checks.append(ValidationIssue("Declining revenue not typical for valuation", "high"))
    elif rev_cagr > 0.25:  # >25% CAGR
        checks.append(ValidationIssue("Very high growth; verify vs. market TAM", "medium"))
    
    # Terminal Value shouldn't exceed 80% of Enterprise Value
    # (indicates model is heavily dependent on terminal value)
    pv_fcf = dcf_model.sum_pv_fcf()
    pv_tv = dcf_model.get_pv_terminal_value()
    tv_pct = pv_tv / (pv_fcf + pv_tv)
    if tv_pct > 0.80:
        checks.append(ValidationIssue("Terminal Value is >80% of EV; model is TV-heavy", "medium"))
    
    return checks
```

## Edge Cases & Failure Modes

### Case 1: Conflicting Data in Multiple Sheets

**Situation**: Revenue in `Comps` sheet shows $100M, but DCF sheet uses $120M

**Handling**:
- package-reader flags inconsistency
- valuation-runner reports: "Data inconsistency: source1=$100M vs. source2=$120M"
- publisher includes in recommendations: "Reconcile revenue assumptions across workbook"

### Case 2: Missing Peer Data

**Situation**: Portfolio Analytics MCP can't find peers for niche company (specialized industrial equipment)

**Handling**:
- valuation-runner notes: "Peer data unavailable; EBITDA margin reasonableness check skipped"
- publisher flags as "Unable to benchmark assumptions"
- recommendation: "Manual peer research needed for margin validation"

### Case 3: Extreme Sensitivity

**Situation**: Model shows ±50% valuation range from reasonable assumption swings (WACC ±1%)

**Handling**:
- valuation-runner flags: "Model exhibits high sensitivity to WACC; terminal value assumption is critical driver"
- publisher recommends: "Stress-test base case assumptions; if WACC increases to 9.5% from 8.5%, value declines 40%"

### Case 4: Circular References in Excel

**Situation**: Revenue growth depends on margin, which depends on revenue

**Handling**:
- package-reader detects circular reference during read
- Output: ValidationError("Circular reference detected in cells...")
- Orchestrator halts; reports error to user

## Escalation Paths

```
┌──────────────────────────────────────┐
│ Valuation Reviewer Completes         │
│ status: completed, findings: ...     │
└────────────────┬─────────────────────┘
                 │
         ┌───────┴────────┐
         │                │
         ▼                ▼
   All Checks Passed    Issues Found
         │                │
         │                ├─→ flag_type: "warning"
         │                │   (assumption outlier)
         │                │   → publisher includes in report
         │                │   → recommend adjustment
         │                │
         │                └─→ flag_type: "error"
         │                    (formula broken)
         │                    → orchestrator halts
         │                    → manual review required
         │
         └──────────────────┘
              │
              ▼
   Next Step: What happens?
   
   If "comprehensive_review": 
     → Results go to LP reporting
     → Publisher creates summary for LP deck
     → Mark valuation "approved for reporting"
   
   If "audit_validation":
     → Results cached for external auditor
     → Publisher creates audit trail
     → Sign-off by CFO required
```

## ความแตกต่างจาก Interactive Plugin

**Valuation Reviewer Plugin** (Interactive):
- User ผ่าน assumptions manually
- Agent ถาม clarifying questions
- Real-time recalculation as user adjusts inputs
- ใช้สำหรับ iterative modeling

**Valuation Reviewer Cookbook** (Managed):
- Fully automated
- Accept complete package as input
- Run comprehensive validation suite
- Output structured report
- ใช้สำหรับ batch validation (50+ portcos in LP portfolio)

## Portfolio Analytics MCP Server

```bash
export PORTFOLIO_MCP_URL="https://your-portfolio-analytics-server"
```

Provides:
- `get_peer_benchmarks()` — WACC median, margin median, growth rates by industry
- `get_valuation_multiples()` — EV/EBITDA, P/E, EV/Revenue by industry/stage
- `get_industry_outlook()` — Sector growth assumptions, margin trends

---

**หมายเหตุ**: Valuation Reviewer ต้องการ well-formed Excel files ที่มี defined formula ranges และ input assumptions cells ที่ชัดเจน
