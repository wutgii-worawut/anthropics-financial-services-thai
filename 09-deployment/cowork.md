---
title: Cowork (Web UI)
parent: 09 การ Deploy
nav_order: 1
---

# Cowork — ติดตั้ง Plugins ใน Workspace

## Cowork คืออะไร

[Cowork](https://cowork.claude.com) คือ Claude workspace — ที่ที่ผู้ใช้ install plugins, create sessions, และ dispatch agents

Cowork ≠ claude.ai (consumer) — มันคือ **business-grade shared workspace** สำหรับ teams ที่อยากให้ Claude ไป่งานกับ firm data + internal tools

## ติดตั้ง Plugins จาก Marketplace

ขั้นที่ 1: ไปที่ Cowork Settings:

```
https://cowork.claude.com → Settings → Plugins → Add plugin
```

ขั้นที่ 2: Paste repo URL:

```
https://github.com/anthropics/claude-for-financial-services
```

Cowork scan repo, ตัดจาก `.claude-plugin/marketplace.json` ด้วยรายชื่อ plugins ทั้งหมด

ขั้นที่ 3: Marketplace list ปรากฏ:

```
financial-analysis    (core modeling)
investment-banking    (deal materials)
equity-research       (earnings, coverage)
private-equity        (sourcing, due diligence)
... [19 more]
```

ขั้นที่ 4: Click toggle ให้เลือกว่า install อะไร

```
☑ financial-analysis          (required first — has all MCPs)
☑ investment-banking
☑ pitch-agent
☑ gl-reconciler
```

ขั้นที่ 5: Confirm

Cowork download plugins, extract skills/commands, wire up MCPs (ถ้าคุณตั้ง env vars/vaults)

## ติดตั้ง Partner Plugins

LSEG + S&P Global plugins มี source ใน `.claude-plugin/marketplace.json`:

```json
{
  "name": "lseg",
  "source": "./plugins/partner-built/lseg",
  "description": "Bond RV, swap curves, FX carry, options vol..."
}
```

Paste same repo URL — marketplace list จะรวม partner plugins ด้วย

ถ้า provider ต้องการ API key (LSEG Refinitiv, S&P Capital IQ):

1. ติดตั้ง plugin
2. ไปที่ plugin settings → "MCP credentials"
3. Paste API key
4. Save

Cowork store credentials ใน vault — ไม่ต่อ network ก่อน key set

## Upload Custom Plugin Zip

ถ้าคุณ fork repo และ customize vertical plugin:

```
cd plugins/vertical-plugins/financial-analysis/
zip -r ~/financial-analysis-custom.zip .
```

ใน Cowork UI:

```
Settings → Plugins → Upload a zip
[drag and drop ~/financial-analysis-custom.zip]
```

Cowork extract, read plugin.json, register skills + commands

## Workspace Management

### องค์กร vs ส่วนบุคคล

- **Organization workspace** — ทีมทั้ง firm share plugins, settings, audit logs
  - 1 admin account
  - multiple member accounts
  - central control ของ MCPs, skills, สิทธิ์

- **Personal workspace** — ของคนเดียว (default)
  - เมื่อ signup, Cowork auto-create personal workspace

### สิทธิ์

Cowork admin ตั้งได้:

- Who can install plugins? (admin only / all members)
- Which MCPs are allowed? (allowlist)
- Which agents are callable? (who can dispatch pitch-agent vs kyc-screener)

## มี Agent ใน Cowork ยังไง

เมื่อคุณ install agent plugin (เช่น `pitch-agent`):

1. Plugin register ตัวเอง ใน marketplace
2. Cowork read plugin.json → ขอ agent manifest
3. Agent skill + commands ready ใน Cowork UI
4. User click "Dispatch pitch-agent" → session opens

User ทำ:

```
"Build a pitch book for Acme Corp, acquisition thesis: operational synergies"
```

Pitch agent (Opus) starts → เรียก skills (comps, precedents, LBO) → 
return branded PowerPoint deck file → Cowork show download button

## ความแตกต่าง vs claude.ai Consumer

| Feature | Cowork (Business) | claude.ai (Consumer) |
|---|---|---|
| Plugins | install จาก marketplace + custom repo | ไม่มี plugins |
| MCPs | support — wire ใน workspace | ไม่รองรับ |
| Agents | named agents (pitch-agent, gl-reconciler, ...) | ไม่มี |
| Audit | session logs, who used what agent | ไม่มี |
| Pricing | usage-based + seat license | per-seat monthly |
| Self-host | ไม่ได้ (Anthropic-hosted) | N/A |
| API dispatch | Managed Agents API (separate) | N/A |

---

**Next:** [Claude Code CLI — install plugins ใน terminal](claude-code.md)
