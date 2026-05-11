---
title: 06 Managed Agent Cookbooks
nav_order: 7
has_children: true
permalink: /06-cookbooks/
---

# Managed Agent Cookbooks

`managed-agent-cookbooks/` — **สูตร (recipe)** สำหรับ deploy agent เดียวกันผ่าน Claude **Managed Agents API**

## ต่างจาก agent-plugins ยังไง?

| มิติ | `plugins/agent-plugins/` | `managed-agent-cookbooks/` |
|------|--------------------------|----------------------------|
| Runtime | Cowork plugin / Claude Code CLI | Managed Agents API |
| ใครรัน | User interactive | API ขับเคลื่อน (batch / scheduled) |
| Format | plugin.json + agents/*.md + skills/ | agent.yaml + orchestrator.md + subagents/ |
| Sub-agents | inline ผ่าน skills | depth-1 subagents (researcher, modeler, writer) |
| Output | conversational | structured JSON schema |

## รายชื่อ Cookbook

ทุก cookbook map 1:1 กับ agent plugin ใน `plugins/agent-plugins/`:

1. [pitch-agent](./pitch-agent.md)
2. [earnings-reviewer](./earnings-reviewer.md)
3. [valuation-reviewer](./valuation-reviewer.md)
4. [model-builder](./model-builder.md)
5. [market-researcher](./market-researcher.md)
6. [meeting-prep-agent](./meeting-prep-agent.md)
7. [kyc-screener](./kyc-screener.md)
8. [statement-auditor](./statement-auditor.md)
9. [gl-reconciler](./gl-reconciler.md)
10. [month-end-closer](./month-end-closer.md)

## โครงสร้างมาตรฐาน

```
managed-agent-cookbooks/<agent>/
├── agent.yaml              # manifest หลัก
├── orchestrator.md         # system prompt ของ orchestrator
├── subagents/
│   ├── researcher.md
│   ├── modeler.md
│   └── writer.md
└── schemas/
    └── handoff.json        # schema สำหรับ output ระหว่าง subagent
```
