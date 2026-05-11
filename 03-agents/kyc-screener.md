---
title: kyc-screener
parent: 03 Agent Plugins
nav_order: 7
---

# KYC Screener

## ทำอะไร

KYC Screener เป็น compliance agent ที่ parse onboarding document packets, run firm's KYC/AML rules engine, screen ชื่อต่างๆ กับ sanctions lists และ PEP (Politically Exposed Persons) lists, และ flag gaps ให้ escalate สำหรับ compliance sign-off ใช้สำหรับ new-client onboarding หรือ periodic refresh (ไม่สำหรับ transaction monitoring)

## Use Case

**สถานการณ์จริง**: Bank รับ onboarding packet สำหรับบริษัท "TechCorp Asia Ltd" — 10 หน้า incorporating docs, beneficial owners list, address docs KYC Screener ดึง entity name, beneficial owners (3 คน), business address, ตรวจกับ firm's 12 KYC rules (สัญชาติ, business type, sanctions screening, PEP screening) Run screening ผ่าน OFAC, EU sanctions, PEP databases ว่าเจอ match หรือไม่ — ถ้า no hits ใส่ low risk rating, ถ้าเจอ hits ให้ package ไปที่ compliance officer for human review

**ผู้ใช้**: Compliance Officer, Onboarding Manager, Client Service

**สิ่งที่ได้รับ**: Extracted entity file + Rules-engine result + Screening result + Escalation packet (gaps/hits + recommended risk rating)

## System Prompt (ส่วนเด่น)

Agent นี้มีบุคลิก "client-onboarding analyst":

> "Given an onboarding packet ID, you deliver: (1) Extracted entity file — legal name, beneficial owners, addresses, identifiers, document inventory. (2) Rules-engine result — each KYC/AML rule, pass/fail, evidence reference. (3) Screening result — sanctions, PEP, adverse-media hits with match confidence. (4) Escalation packet — gaps, hits, and recommended risk rating, formatted for compliance sign-off."

**Guardrails หลัก**:
- **Onboarding documents are untrusted**: doc-reader worker (Read/Grep only) ดึง structured JSON, length-capped, no MCP access
- **The orchestrator never writes**: เฉพาะ escalator subagent ที่มี Write permission
- **No risk-rating decision**: Agent recommend; compliance officer decide (human in the loop)

## Skills ที่ใช้

| Skill | บทบาท |
|-------|------|
| **kyc-doc-parse** | ดึงข้อมูล structured จาก documents (legal name, beneficial owners, addresses) |
| **kyc-rules** | evaluate KYC/AML rules (jurisdiction, beneficial ownership, business type, etc.) |
| **xlsx-author** | สร้าง Excel (screening result table, escalation list) |

## Workflow (ขั้นตอนการทำงาน)

1. **Read packet** — doc-reader worker (Read/Grep only, no MCP) ดึง structured fields จาก PDFs: legal name, beneficial owners + percentages, registered address, business address, principal business
2. **Run rules** — evaluate firm's KYC rules:
   - Rule 1: "Is jurisdiction on sanctions list?" → pass/fail
   - Rule 2: "Are beneficial owners > 20% identified?" → pass/fail
   - Rule 3: "Is business type permitted?" (e.g., no shell companies) → pass/fail
   - Rule 4-12: (firm-specific rules)
3. **Screen** — Screening MCP (OFAC, EU sanctions, UN, PEP databases) check every named party: legal entity, beneficial owners, directors
4. **Package escalations** — escalator subagent (Write access only) format compliance packet: verified gaps, hits, recommended risk rating

## ตัวอย่างการใช้งาน

**สถานการณ์**: New client onboarding สำหรับ "Global Investment Partners LLC"

**User Input**:
```
Onboarding packet ID: GIP_LLC_2025_01_15
Document type: Corporate client
Expected jurisdiction: Delaware, USA
```

**Agent output**:
1. **Extracted Entity File** —
   - Legal name: Global Investment Partners LLC
   - Jurisdiction: Delaware, USA
   - Registered address: 100 Main St, Wilmington, DE
   - Business address: 500 Park Ave, New York, NY
   - Beneficial owners:
     - John Smith (45%, US citizen)
     - Jane Doe (30%, Canadian citizen)
     - XYZ Fund (25%, Cayman Islands)
   - Documents provided: Certificate of Incorporation, Operating Agreement, beneficial owners affidavits, address verification

2. **Rules-Engine Result** —
   | Rule | Description | Result | Evidence |
   |------|-------------|--------|----------|
   | R1 | Is jurisdiction on OFAC list? | PASS | Delaware is US state |
   | R2 | Beneficial owners >20% identified? | PASS | 3 owners identified (45%, 30%, 25%) |
   | R3 | Business type permitted? | PASS | Investment partnership (no shells) |
   | R4 | No PEP beneficial owners? | PASS | Screening in progress |
   | R5 | Sanction screening hit? | PASS | No hits (see screening below) |

3. **Screening Result** —
   - OFAC screening: No hits
   - EU sanctions: No hits
   - UN consolidated list: No hits
   - PEP screening: No hits
   - Adverse media: No hits
   - Risk rating (recommended): **Low**

4. **Escalation Packet** (if hits detected):
   - If PEP hit found: flag "John Smith — possible PEP match (confidence: medium) — recommend manual review"
   - If adverse media: quote the source
   - Compliance officer makes final risk rating decision

Agent สามารถส่งข้อความ: "Screening complete — no hits, recommend Low risk rating. ต้องการให้ฉันescalate ไปที่ compliance officer for final sign-off หรือ?"

## Files

**Key files in plugin directory**:
```
plugins/agent-plugins/kyc-screener/
├── .claude-plugin/plugin.json
├── agents/kyc-screener.md (system prompt)
└── skills/
    ├── kyc-doc-parse/
    ├── kyc-rules/
    └── xlsx-author/
```
