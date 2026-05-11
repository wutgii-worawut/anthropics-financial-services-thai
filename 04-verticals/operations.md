---
title: Operations (ปฏิบัติการ)
parent: 04 Vertical Plugins
nav_order: 7
---

# Operations — ประตูคนเข้าใจม

**Operations** vertical ช่วย operations teams, compliance officers, KYC/AML specialists ประเมิน customer documentation, verify regulatory compliance, manage onboarding workflows

## ภาพรวม

เป้าหมาย: Automate KYC/AML workflows — document parsing, rules evaluation, compliance checking

**ผู้ใช้:**
- Operations teams
- Compliance officers
- KYC/AML specialists
- Client onboarding teams
- Risk teams (client assessment)

**Workflows ที่สนับสนุน:**
- KYC document parsing (ID verification, beneficial ownership, source of funds)
- AML rules evaluation (regulatory screening, PEP checks, sanctions lists)
- Client assessment (risk profile scoring, approval/escalation)
- Onboarding workflow management (document checklist, approval hierarchy)

## Commands (Slash Commands)

*Note: Operations ไม่มี `/commands` — ใช้ skills เรียกจาก KYC document uploads*

**Skills invoked via document upload:**

| Skill | ใช้ได้อย่างไร |
|-------|---------|
| `/kyc-doc-parse` | Upload KYC documents (ID, articles, ownership structure) → Extract key information |
| `/kyc-rules` | Evaluate client against rules grid (regulatory, sanctions, PEP) → Risk assessment |

### ตัวอย่างการใช้ Commands

**KYC Document Parsing:**
```
Upload: Client KYC package
- Government ID (passport, driver license)
- Articles of incorporation or bylaws
- Beneficial ownership register
- Source of funds documentation
- Bank references

Output:
- Extracted client name, ID number, DOB, nationality
- Company registration info, business address
- UBO (ultimate beneficial owner) identified: John Smith (75% ownership)
- Funds sourcing: "Asset sale proceeds, deposited via wire"
- All required docs: ✓ Present
```

**KYC Rules Evaluation:**
```
Upload: Client profile + rules grid
Client: "Apex Consulting Inc", UBO John Smith, nationality: [Country X]

Evaluate against rules:
✗ SANCTIONS: [Country X] on OFAC list → ESCALATE (cannot on-board)
✓ PEP: John Smith not on PEP list
✓ AML: No red flags in source of funds
✓ REGULATORY: Company properly licensed in home jurisdiction

Recommendation: ESCALATE TO COMPLIANCE OFFICER
Reason: Country-based sanction (OFAC), cannot on-board this client
```

## Skills ที่มี

| ชื่อ Skill | ทำอะไร | วิธีหลัก |
|-----------|---------|---------|
| `kyc-doc-parse` | Extract information from KYC documents: ID, beneficial ownership, company structure, fund sources | OCR + entity extraction; identifies key fields (name, DOB, UBO %, sources) |
| `kyc-rules` | Evaluate client against compliance rules grid: sanctions, PEP, AML risk factors | Rules engine; applies regulatory requirements (OFAC, PEP lists, AML red flags) |

## Hooks

**File:** `/operations/hooks/hooks.json`

Status: **No hooks** — Skills are invoked on-demand via document upload

## MCP Connectors ที่เกี่ยวข้อง

| Connector | ใช้สำหรับ |
|-----------|---------|
| **Sanctions lists** (OFAC, UN, EU, UK) | Check client nationality, UBO nationality against sanctions |
| **PEP databases** | Verify client/UBO not on politically exposed person list |
| **AML monitoring** | Monitor for suspicious activity patterns |
| **Regulatory databases** | Verify licenses, registration status |

## ความสัมพันธ์กับ Agent Plugins

**Operations** standalone — does not consume from other verticals
(Provides compliance clearance that enables PE/IB/Wealth workflows)

## ตัวอย่างการใช้งาน

### Workflow: New Client Onboarding (1 hour)

```
OPERATIONS TEAM receives new client application:
Company: "Global Growth Fund III" (PE fund)
Requesting account: $100M investment account
Contact: Jane Doe (Chief Investment Officer)

STEP 1: Document Collection
Operations team sends KYC package request:
- Management agreement (fund governing docs)
- Certificate of formation / bylaws
- LP list with beneficial ownership percentages
- Source of funds documentation (e.g., capital raise letter from LPs)
- Bank references (2 minimum)
- ID for authorized signatories (Jane Doe)
- FATCA/W-9 certification

STEP 2: Document Review & Parsing
Client returns completed KYC package

Operations team uploads to /kyc-doc-parse:
- 3 GB folder with 15 documents (PDF, Word, Excel)

Skill extracts:
COMPANY INFORMATION:
- Legal name: Global Growth Fund III, LP
- Entity type: Limited Partnership
- Jurisdiction: Delaware
- Formation date: 2023-01-15
- Registration #: LP-2023-00456

AUTHORIZED SIGNATORIES:
- Jane Doe (CIO): US Passport #123456789, DOB 1975-03-10, nationality US
- David Chen (CFO): US Passport #987654321, DOB 1980-07-22, nationality US

BENEFICIAL OWNERS (LP list):
- Cornerstone Pension Fund (35%, US domiciled)
- ABC Family Office (25%, UK domiciled)
- DEF Endowment (20%, US domiciled)
- XYZ Foundation (15%, Switzerland domiciled)
- Management team (5%, US domiciled)

SOURCE OF FUNDS:
"Committed capital from LP base; initial funding from Cornerstone Pension capital call (documented letter)"

STATUS: All required documents ✓ Present, readable, complete

STEP 3: Rules Evaluation
Operations team uploads to /kyc-rules:
Client profile + rules grid

Evaluate against rules:
✓ SANCTIONS CHECK:
  - Client (Delaware US) → Not on OFAC/UN/EU sanctions lists
  - UBOs: Cornerstone (US), ABC (UK), DEF (US), XYZ (Switzerland), Management (US)
  - No sanctions flag

✓ PEP CHECK:
  - Jane Doe (CIO) → Not on PEP list
  - David Chen (CFO) → Not on PEP list
  - No PEP flag

✓ AML RISK ASSESSMENT:
  - Fund purpose: Investment management (standard, low-risk activity)
  - Source of funds: Pension/endowment commitments (legitimate sources)
  - Transaction size: $100M (consistent with stated AUM)
  - No AML red flags

✓ REGULATORY CHECK:
  - Fund licensed as SEC-regulated investment manager
  - All LP sources US/UK/Switzerland domiciled (no high-risk jurisdictions)
  - No compliance issues

RECOMMENDATION: APPROVED FOR ON-BOARDING
Risk Rating: LOW
Expected approval: 1-2 business days (final legal review)

STEP 4: Approval & Setup
Compliance officer reviews /kyc-rules output:
- All checks passed ✓
- No escalations needed
- Approved for on-boarding

Operations team:
1. Creates account in core system
2. Sends welcome letter + account setup instructions
3. Client begins investment activity

Total time: 1 hour (mostly automated via skills; compliance review = 20 min)
```

### Workflow: High-Risk Client Escalation (1.5 hours)

```
OPERATIONS TEAM receives new client application:
Company: "Prosperity Trading Company Ltd"
Requested account: $50M trading account
Jurisdiction: [Sanctioned Country]

STEP 1: Document Upload & Parsing
Operations team uploads KYC documents to /kyc-doc-parse

Extracted information:
- Legal name: Prosperity Trading Company Ltd
- Jurisdiction: [Country X] (domiciled)
- UBO: Mohammed Al-[Name], nationality [Country X]
- Source of funds: "Asset sales from real estate portfolio"
- Directors: 3 individuals, all [Country X] nationals

STEP 2: Rules Evaluation — RED FLAG
Operations team uploads to /kyc-rules

Evaluate against rules:
✗ SANCTIONS ALERT:
  - Client jurisdiction [Country X] is on OFAC Specially Designated Nationals (SDN) list
  - Cannot on-board clients domiciled in sanctioned countries
  
✗ AML RISK ESCALATION:
  - Vague source of funds ("asset sales") — requires documentation
  - UBO nationality matches sanctioned country
  - Trading activity (common for AML risk)
  
RECOMMENDATION: ESCALATE TO COMPLIANCE OFFICER
Risk Rating: HIGH
Action Required: COMPLIANCE REVIEW & APPROVAL NEEDED BEFORE PROCEEDING

STEP 3: Compliance Officer Review (30 min)
Compliance officer reviews /kyc-rules escalation report:
- Identifies: Sanctioned country jurisdiction
- Action: Check if client qualifies for any OFAC licenses or exemptions (unlikely for this entity)
- Decision: Cannot on-board without OFAC license

Compliance officer actions:
1. Document escalation reason in client file
2. Send client rejection letter (citing sanctions compliance)
3. Close application

Outcome: Client APPLICATION DENIED due to sanctions compliance
Total process: 1.5 hours (automated alerts saved time; compliance officer handles final decision)
```

### Workflow: Beneficial Ownership Verification (45 min)

```
OPERATIONS TEAM processes client with complex ownership:
Company: "International Holdings B.V." (Dutch company)
Requested account: $200M investment account

KYC Package includes:
- Articles of Association (Dutch)
- Shareholder register (shows intermediate holding company, not final UBO)
- Bank references

STEP 1: Initial Parse
Operations team uploads to /kyc-doc-parse

Extracted:
- Legal name: International Holdings B.V.
- Shareholder: "Global Investment Partners LTD" (Cayman Islands)
  [Intermediate owner, not ultimate beneficial owner]
- No direct UBO documented

RED FLAG: UBO chain incomplete

STEP 2: Document Request
Compliance requires further documentation:
- Shareholder register for Global Investment Partners LTD (Cayman)
- Beneficial ownership certification for Global Investment Partners
- Personal ID for any natural person with >10% beneficial ownership

Client provides additional docs (2-day turnaround)

STEP 3: Re-parse with Updated Docs
Operations uploads additional docs to /kyc-doc-parse

Now extracted:
- Direct owner: Global Investment Partners LTD (Cayman)
- Ultimate owner: Sophia Chen (Hong Kong, 100% beneficial owner)
- Personal ID provided: HK Passport #123456

Complete ownership chain verified:
International Holdings B.V. (Dutch) → 100% owned by Global Investment Partners LTD (Cayman) → 100% owned by Sophia Chen (Hong Kong)

STEP 4: Rules Evaluation
Operations team uploads complete profile to /kyc-rules

Evaluate:
✓ SANCTIONS: Sophia Chen (Hong Kong national) — not on OFAC list
✓ PEP: Sophia Chen — not on PEP list
✓ AML: UBO verified, source of funds legitimate
✓ REGULATORY: All entities properly licensed/registered

RECOMMENDATION: APPROVED FOR ON-BOARDING
Risk Rating: LOW (with full beneficial ownership documentation)

Outcome: Account opens after complete UBO verification
Total time: 45 min (parsing + rules check); 2-day wait for documentation
```

## Files

| Path | ทำอะไร |
|------|---------|
| `skills/kyc-doc-parse/SKILL.md` | KYC document extraction |
| `skills/kyc-rules/SKILL.md` | Rules grid evaluation |
| `.claude-plugin/plugin.json` | Plugin metadata |

---

**Operations Vertical = Gatekeeper**: KYC/AML compliance ensures all counterparties are cleared before investment flows
