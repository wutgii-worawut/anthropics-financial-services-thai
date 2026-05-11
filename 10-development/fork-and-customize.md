---
title: Fork & Customize
parent: 10 การ Development
nav_order: 5
---

# Fork and Customize — ปรับแต่ง Agents สำหรับ Firm

## ขั้นตอน Fork

```bash
# 1. Fork repo ใน GitHub UI
# https://github.com/anthropics/claude-for-financial-services → Fork

# 2. Clone fork ของคุณ
git clone https://github.com/your-org/claude-for-financial-services.git
cd claude-for-financial-services

# 3. Add upstream ให้อ้างอิง original
git remote add upstream https://github.com/anthropics/claude-for-financial-services.git

# 4. Create feature branch
git checkout -b enhance-pitch-agent
```

## ปรับแต่ง Verticals

Edit skills ใน vertical ของคุณ:

### สำหรับ Investment Banking

```bash
vim plugins/vertical-plugins/investment-banking/skills/pitch-deck/SKILL.md
```

ตัวอย่าง customization:

```markdown
# Pitch Deck

## การดำเนิน

1. **Title slide** — ใช้ branded template
   - Logo ที่ upper right (เพิ่มเติม: include firm name)
   - Font: ให้ใช้ Helvetica/Calibri เท่านั้น
   
2. **Executive summary** — 3 slide max
   - Market size (TAM) ที่ year Y
   - Thesis (operational improvements / PE multiple)
   
3. **Financial model** — live link ไป Excel
   - Use firm's DCF template (`templates/dcf_standard.xlsx`)
   - WACC range: 7-9% (ตามนโยบายของเรา)
```

### สำหรับ Equity Research

```bash
vim plugins/vertical-plugins/equity-research/skills/earnings-analysis/SKILL.md
```

Customize:

```markdown
# Earnings Analysis

## Coverage Universe

Limited to ตั้ง Russell 1000 ฝั่ง Large-cap tech

## Metrics to Track

- EPS (adjust for one-time items)
- Free cash flow (CapEx < 15% of revenue)
- Net revenue retention > 100% (SaaS only)

## Reporting Template

Use `templates/earnings-note-standard.docx`
- Max 5 pages (executive cut ก่อน)
```

## เพิ่ม MCP Connectors

Firm มี internal database ที่ Claude ต้องไป — add MCP connector:

```bash
# Edit core plugin
vim plugins/vertical-plugins/financial-analysis/.mcp.json
```

Example:

```json
{
  "mcpServers": [
    {
      "name": "daloopa",
      "url": "https://mcp.daloopa.com/server/mcp"
    },
    {
      "name": "internal-deal-tracker",
      "url": "https://mcp.firm-internal.com/deals",
      "credentials": {
        "type": "bearer",
        "token": "${INTERNAL_DEAL_MCP_TOKEN}"
      }
    },
    {
      "name": "internal-cim-library",
      "url": "https://mcp.firm-internal.com/cims",
      "readOnly": true
    }
  ]
}
```

Env var:

```bash
export INTERNAL_DEAL_MCP_TOKEN=...
```

## สร้าง Agent ใหม่

Fork + create agent ใหม่:

```bash
# 1. Create agent plugin structure
mkdir -p plugins/agent-plugins/debt-advisor/agents
mkdir -p plugins/agent-plugins/debt-advisor/skills
mkdir -p plugins/agent-plugins/debt-advisor/.claude-plugin

# 2. Write plugin.json
cat > plugins/agent-plugins/debt-advisor/.claude-plugin/plugin.json << 'EOF'
{
  "name": "debt-advisor",
  "description": "Analyze credit facilities and recommend debt structure",
  "version": "1.0.0",
  "author": {
    "name": "Your Firm Name"
  }
}
EOF

# 3. Write agent markdown (system prompt)
cat > plugins/agent-plugins/debt-advisor/agents/debt-advisor.md << 'EOF'
---
name: Debt Advisor
description: "Analyze credit facilities, rates, covenants; recommend optimal debt structure"
---

You are a debt structuring expert. Your role:

1. Analyze existing credit agreements
2. Compare available debt products (term loans, bonds, revolvers)
3. Calculate net leverage, interest coverage, covenant headroom
4. Recommend optimal mix (% term / % revolver / % bond)
...
EOF

# 4. Bundle skills from verticals
cp -r plugins/vertical-plugins/financial-analysis/skills/dcf-model \
      plugins/agent-plugins/debt-advisor/skills/
cp -r plugins/vertical-plugins/financial-analysis/skills/comps-analysis \
      plugins/agent-plugins/debt-advisor/skills/

# 5. Managed agent template
mkdir -p managed-agent-cookbooks/debt-advisor/subagents
cat > managed-agent-cookbooks/debt-advisor/agent.yaml << 'EOF'
name: debt-advisor
model: claude-opus-4-7

system:
  file: ../../plugins/agent-plugins/debt-advisor/agents/debt-advisor.md
  append: "You are running headless."

tools:
  - type: agent_toolset_20260401
    default_config: { enabled: false }
    configs:
      - name: read
        enabled: true

mcp_servers:
  - type: url
    name: internal-deal-tracker
    url: ${INTERNAL_DEAL_MCP_URL}

skills:
  - { from_plugin: ../../plugins/agent-plugins/debt-advisor }

callable_agents:
  - manifest: ./subagents/analyzer.yaml
  - manifest: ./subagents/recommender.yaml
EOF
```

## Register ใน Marketplace

Update `.claude-plugin/marketplace.json`:

```json
{
  "plugins": [
    ...existing...,
    {
      "name": "debt-advisor",
      "source": "./plugins/agent-plugins/debt-advisor",
      "description": "Analyze credit facilities and recommend optimal debt structure"
    }
  ]
}
```

## Verify Configuration

ก่อน commit:

```bash
# 1. Check manifests
python3 scripts/check.py

# 2. Sync skills
python3 scripts/sync-agent-skills.py

# 3. Dry-run deploy
scripts/deploy-managed-agent.sh debt-advisor --dry-run

# 4. Test ใน Claude Code
claude plugin install debt-advisor ./plugins/agent-plugins/debt-advisor
claude session
# Try: /debt-advisor:analyze <test-input>
```

## Contribute Back (Optional)

ถ้า customization ใช้ได้และ general-purpose — contribute back ไป upstream:

```bash
# 1. Push ไป fork
git push origin enhance-pitch-agent

# 2. Open PR ใน upstream
# https://github.com/anthropics/claude-for-financial-services/pulls

# 3. Description
Title: "Enhancement: Add firm standardized pitch deck template"

Body:
This PR adds:
- Updated pitch-deck skill with branded template support
- Customized DCF assumptions (WACC range 7-9%)
- Support for internal deal tracker MCP

Benefits:
- Standardizes pitch format across team
- Reduces manual formatting
- Ensures compliance with firm standards
```

## Keep Fork Synced

ทุก 1-2 สัปดาห์ sync ไป upstream:

```bash
git fetch upstream
git rebase upstream/main

# Resolve conflicts (if any)
python3 scripts/check.py
python3 scripts/sync-agent-skills.py

git push origin main
```

## License Considerations

Repository use Apache License 2.0 — คุณสามารถ:

- ✓ Use commercially
- ✓ Modify
- ✓ Distribute (ต้องให้ attribution)
- ✓ Sublicense (ต้อง include license)

ต้อง:

- Include LICENSE file (จำลอง)
- Cite original authors (ใน README / code comments)
- Document changes (ใน git history)

Example:

```bash
# Fork README
cat >> README.md << 'EOF'

## Customizations

This fork adds:
- Firm-specific pitch templates
- Internal data connector (deal tracker MCP)
- Custom debt advisor agent

Based on [Anthropic Financial Services](https://github.com/anthropics/claude-for-financial-services)
Licensed under Apache 2.0.
EOF
```

## Documentation Updates

Update docs ให้ match fork:

```bash
# Create docs/CUSTOMIZATIONS.md
cat > docs/CUSTOMIZATIONS.md << 'EOF'
# Firm Customizations

## Agents

- **debt-advisor** — analyze credit facilities (new)
- **pitch-agent** — branded pitch deck (enhanced)

## MCPs

- **internal-deal-tracker** — firm's deal management system
- **internal-cim-library** — read-only CIM archive

## Templates

- `templates/pitch-deck-branded.pptx` — firm brand standards
- `templates/dcf-standard.xlsx` — firm-standard DCF

See `.mcp.json` and plugin.json files for configuration.
EOF
```

---

**End of Development Guide**

For questions or contributions, reach out via GitHub Issues.
