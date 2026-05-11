---
title: Claude Code CLI
parent: 09 การ Deploy
nav_order: 2
---

# Claude Code CLI — ติดตั้ง Plugins ใน Terminal

## Claude Code คืออะไร

Claude Code (`claude` CLI) คือ terminal tool ให้ developer install plugins locally, run agents headlessly, debug skill execution, and browse marketplace

ติดตั้งมาคู่กับ Claude IDE — หรือ download จาก https://www.anthropic.com/claude/code

## Add Marketplace

ขั้นแรก register marketplace ของ repo:

```bash
claude plugin marketplace add anthropics/claude-for-financial-services \
  https://github.com/anthropics/claude-for-financial-services
```

Output:

```
✓ Added marketplace: claude-for-financial-services
  Registry: https://github.com/anthropics/claude-for-financial-services
  Plugins: 20 available
```

## Install Plugins

ตอนนี้ install ตัวเลือก plugins:

```bash
# Core financial modeling + all MCPs
claude plugin install financial-analysis@claude-for-financial-services

# Vertical skill bundles
claude plugin install investment-banking@claude-for-financial-services
claude plugin install equity-research@claude-for-financial-services

# Named agents
claude plugin install pitch-agent@claude-for-financial-services
claude plugin install gl-reconciler@claude-for-financial-services
```

Syntax: `claude plugin install <name>@<marketplace>`

## ใช้ CLI Commands

หลังจากติดตั้ง skills และ agents พวกมันให้ slash commands:

```bash
claude session                # start interactive session
```

ใน session ทำได้:

```
User: "Run a comps analysis for Microsoft"

Claude: /financial-analysis:comps
(calls skill: comps-analysis)
(returns trading multiples, peer analysis)
```

หรือ explicit:

```
User: "/financial-analysis:dcf Microsoft, WACC=8%, Terminal=3%"
```

## Dispatch Agents Headlessly

ถ้า Claude Code embed ใน workflow (CI/CD, Airflow, cron):

```bash
claude agent dispatch pitch-agent \
  --input "Build pitch: Acme Corp, thesis: margin expansion" \
  --wait
```

Output:

```json
{
  "status": "complete",
  "files": [
    {
      "path": "./pitch-deck-acme.pptx",
      "size": "2.4MB"
    }
  ]
}
```

## Debug Plugin Execution

Test skill locally:

```bash
claude plugin debug financial-analysis:comps \
  --input "Comps for Apple in software" \
  --verbose
```

Output show:

```
[Step 1] Comps skill triggered
  - Query normalized: "Apple software comparables"
  - Peer list: MSFT, GOOGL, NVDA, META
[Step 2] Fetching trading multiples (via MCP)
  - P/E: [18, 22, 25, 28]
  - EV/EBITDA: [8.5, 10.2, 12.1, 14.3]
[Step 3] Analysis
  - Apple median P/E: 22.0
  - Sector median: 21.3
  - Conclusion: In-line with peers
```

## Local vs Remote MCPs

ทั้ง Cowork และ Claude Code รองรับ MCPs — แต่ destination ต่างกัน:

### Cowork
- MCPs managed centrally ใน workspace
- Admin configure credentials หนึ่งครั้ง
- All members share same MCP URLs

### Claude Code
- MCPs run locally ใน your machine
- Set `DALOOPA_API_KEY`, `FACTSET_API_KEY`, ... ใน `~/.claude/env`
- Test ก่อนแล้ว push ไปยัง Cowork

## Environment Setup

Create `~/.claude/env` ให้ set credentials:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
export DALOOPA_API_KEY=...
export FACTSET_API_KEY=...
export MORNINGSTAR_API_KEY=...
export LSEG_REFINITIV_KEY=...
```

Load on startup:

```bash
source ~/.claude/env && claude session
```

## Custom/Forked Plugins

ถ้าคุณ fork financial-services repo และเพิ่ม custom skills:

```bash
# Install จาก your repo
claude plugin install my-banking-extensions \
  https://github.com/yourorg/financial-services-fork

# Or via local path (for dev)
claude plugin install my-extensions ./plugins/vertical-plugins/investment-banking
```

Run CLI test:

```bash
claude plugin debug my-extensions:custom-skill \
  --input "test input" \
  --verbose
```

## Plugin Marketplace Search

List all available plugins:

```bash
claude plugin list --marketplace claude-for-financial-services
```

Output:

```
Name                    Version  Description
financial-analysis      1.0.0    Core modeling + MCPs
investment-banking      1.0.0    Deal materials
equity-research         1.0.0    Earnings & coverage
...
```

Search:

```bash
claude plugin search "comps" --marketplace claude-for-financial-services
```

Result:

```
financial-analysis — has /comps command
investment-banking — uses comps-analysis skill
equity-research — references peer-comps
```

## Build Custom CLI Tool

Embed Claude + plugins ใน Python script:

```python
from anthropic import Anthropic
from claude_plugins import load_marketplace

client = Anthropic()
marketplace = load_marketplace("anthropics/claude-for-financial-services")

# Install plugins programmatically
marketplace.install("financial-analysis")
marketplace.install("pitch-agent")

# Dispatch agent
response = client.agents.dispatch("pitch-agent", input={
    "target": "Acme Corp",
    "thesis": "EBITDA margin expansion via cloud migration"
})

print(f"Pitch deck: {response.files[0].path}")
```

---

**Next:** [Managed Agents API — deploy agents ใน production](managed-agents.md)
