---
title: Claude Plugin Marketplace
parent: 02 โครงสร้างโฟลเดอร์
nav_order: 2
---

# Claude Plugin Marketplace Configuration

ที่อยู่: `.claude-plugin/marketplace.json`

ไฟล์นี้เป็นศูนย์กลางในการลงทะเบียน plugins ทั้งหมด และบอก Claude Cowork วิธีอ้างอิงและค้นหา plugins ในกลุ่มนี้

## marketplace.json — โครงสร้าง

```json
{
  "name": "claude-for-financial-services",
  "owner": {"name": "Matt Piccolella"},
  "plugins": [...]
}
```

### ส่วนประกอบ

| Field | ค่า | ความหมาย |
|---|---|---|
| `name` | `claude-for-financial-services` | ชื่อมหาวิทยาลัย (marketplace name) |
| `owner.name` | `Matt Piccolella` | ผู้แต่ง/maintainer |
| `plugins` | array of objects | รายการ plugins ทั้งหมด |

## Plugin Entries

ทั้งหมด **20 plugins** ลงทะเบียน:

### Vertical Plugins (8 แกน + 2 partner)

```json
{
  "name": "financial-analysis",
  "source": "./plugins/vertical-plugins/financial-analysis",
  "description": "Core financial modeling and analysis tools: DCF, comps, LBO, 3-statement models, competitive analysis, and deck QC"
}
```

**Verticals**:
1. `financial-analysis` — core (modeling, comps, DCF, LBO, 3-statement, Excel audit)
2. `investment-banking` — deal materials (CIMs, teasers, pitch decks, buyer lists, merger models)
3. `equity-research` — coverage (earnings, initiations, model updates)
4. `private-equity` — sourcing & portfolio (DD, IC memos, value creation)
5. `wealth-management` — advisors (client reviews, financial plans, rebalancing)
6. `fund-admin` — operations (GL recon, accruals, NAV tie-out)
7. `operations` — compliance (KYC screening, rules grids)
8. `lseg` — *(partner)* — bonds, curves, FX carry, vol analytics
9. `sp-global` — *(partner)* — tear sheets, earnings previews, funding digests

### Agent Plugins (10 ตัวแทน)

```json
{
  "name": "pitch-agent",
  "source": "./plugins/agent-plugins/pitch-agent",
  "description": "Comps, precedents, LBO to a branded pitch deck, end to end"
}
```

**Agents**:
1. `pitch-agent` — Coverage & advisory
2. `market-researcher` — Research & modeling
3. `earnings-reviewer` — Research & modeling
4. `meeting-prep-agent` — Coverage & advisory
5. `model-builder` — Research & modeling
6. `gl-reconciler` — Fund admin & finance ops
7. `kyc-screener` — Operations & onboarding
8. `valuation-reviewer` — Fund admin & finance ops
9. `month-end-closer` — Fund admin & finance ops
10. `statement-auditor` — Fund admin & finance ops

### Special Plugin (1)

```json
{
  "name": "claude-for-msft-365-install",
  "source": "./claude-for-msft-365-install",
  "description": "Provision direct cloud access (Vertex AI, Bedrock, or LLM gateway) for the Claude Microsoft 365 add-in..."
}
```

## วิธี Cowork ใช้ marketplace.json

### ตัวอย่างการไหล (Flow)

1. **User ใน Cowork** → **Settings → Plugins → Add plugin**
2. User วาง repo URL: `https://github.com/anthropics/claude-for-financial-services`
3. **Cowork reads** `.claude-plugin/marketplace.json`
4. **Marketplace discovers**:
   - ชื่อมหาวิทยาลัย: "claude-for-financial-services"
   - Plugins ทั้ง 20 รายการพร้อมเส้นทาง source

### UI Presentation

Cowork จะแสดงรายการเลือก:
```
claude-for-financial-services (Matt Piccolella)
├─ financial-analysis (core)
├─ investment-banking
├─ equity-research
├─ ...
├─ pitch-agent
├─ market-researcher
├─ ...
└─ claude-for-msft-365-install
```

User สามารถเลือกเฉพาะ plugins ที่ต้องการ

### CLI Equivalent

```bash
# Add marketplace
claude plugin marketplace add anthropics/claude-for-financial-services

# Install specific plugins
claude plugin install financial-analysis@claude-for-financial-services
claude plugin install pitch-agent@claude-for-financial-services
```

## Plugin Discovery Process

```
marketplace.json
    ↓
    Cowork reads: name, owner, plugins[]
    ↓
    For each plugin entry:
    - name → display name
    - source → relative path to that plugin's root
    - description → tooltip in UI
    ↓
    Cowork locates: plugins/agent-plugins/pitch-agent/
                or plugins/vertical-plugins/financial-analysis/
    ↓
    Reads: plugin.json (metadata)
    ↓
    Extracts: commands, skills, manifest info
    ↓
    User installs → added to Cowork session
```

## Relationship between marketplace.json and plugin.json

| File | Location | ระดับ | ความหมาย |
|---|---|---|---|
| `marketplace.json` | `.claude-plugin/marketplace.json` | **Collection** | มหาวิทยาลัยทั้งหมด 20 plugins |
| `plugin.json` | `plugins/<type>/<slug>/.claude-plugin/plugin.json` | **Individual** | Metadata สำหรับ plugin หนึ่ง |

**ตัวอย่าง**:

marketplace.json: `"source": "./plugins/agent-plugins/pitch-agent"`
↓ Cowork opens: `plugins/agent-plugins/pitch-agent/.claude-plugin/plugin.json`
↓ Reads: name, description, version, commands, skills

---

## เหตุการแยก Marketplace + Plugin.json

1. **Marketplace** = **catalog** — บอกว่า plugins อะไรมีอยู่
2. **Plugin.json** = **manifest** — บอกว่า plugin นั้นทำงานไหน

เมื่ออัปเดต plugin หนึ่งชิ้น → อัปเดต `plugin.json` ใน folder นั้นๆ
Marketplace.json ไม่เปลี่ยน (เพราะมันเพียงแค่อ้างอิง relative paths)

---

**สรุป**: marketplace.json คือ "แค็ตตาล็อก" ที่ Cowork ใช้ค้นหาและติดตั้ง plugins 20 รายการจาก repository นี้ — ทั้งแนวตั้ง (verticals), ตัวแทน (agents), และ special tools
