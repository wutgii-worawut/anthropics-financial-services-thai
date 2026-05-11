---
title: ไฟล์ระดับรูท (Root Files)
parent: 02 โครงสร้างโฟลเดอร์
nav_order: 1
---

# ไฟล์ระดับรูท

ไฟล์ที่อยู่ในรูทของ repository ได้แก่: `README.md`, `CLAUDE.md`, `LICENSE`, และ `.gitignore` — ซึ่งกำหนดวัฒนาการ คุณสมบัติ และการจัดการของโครงการ

## README.md — คำแนะนำหลัก (16 KB)

`README.md` เป็นจุดเข้าแรกสำหรับผู้ใช้ โครงสร้างของมันประกอบด้วย:

### โครงสร้างเนื้อหา

1. **บทนำ** — ตัวแทน (agents) พร้อม skills และ data connectors สำหรับ workflow การเงินทั่วไป เช่น investment banking, equity research, private equity, wealth management

2. **ความสำคัญ** — การแจ้งเตือน: โค้ดที่นี่เป็น **draft work product** เท่านั้น ต้องตรวจสอบและลงนามโดยผู้มีคุณสมบัติก่อนใช้งาน ไม่ได้ทำให้คำแนะนำด้านการลงทุนหรือทำธุรกรรม

3. **รายการสิ่งที่อยู่ใน Repo** — 
   - Agents: 10 ตัวแทนที่มีชื่อ (Pitch Agent, Market Researcher, GL Reconciler, เป็นต้น)
   - Vertical Plugins: มัดทักษะ (skills) และ connector โดยกลุ่มธุรกิจแนวตั้ง (financial-analysis, investment-banking, equity-research, ...)
   - Scripts: โครงสร้างมาตรฐาน

4. **ตารางตัวแทน** — แสดงชื่อ, ฟังก์ชันการทำงาน, และวิธีใช้งาน

5. **เริ่มต้นใช้งาน** — 
   - **Cowork**: วิธีติดตั้งจาก plugin marketplace
   - **Claude Code**: คำสั่ง `claude plugin` สำหรับติดตั้ง
   - **Claude Managed Agents API**: วิธี deploy ผ่าน `/v1/agents`

6. **How It Fits Together** — ตารางอธิบายระบบ:
   - Agents = ปลายน้อย (endpoint) สำเร็จรูป
   - Skills = ความรู้เฉพาะที่ใช้ร่วมกัน
   - Commands = slash actions (`/comps`, `/dcf`, etc.)
   - Connectors = MCP servers (Model Context Protocol)
   - Managed-agent wrappers = `agent.yaml` สำหรับ headless deployment

7. **Vertical Plugins** — ตารางทั้ง 9 plugins พร้อมคำอธิบายและตัวอักษร "(core)" หรือ "(partner)"

8. **MCP Integrations** — 11 ตัวให้บริการข้อมูล (Daloopa, Morningstar, S&P Global, FactSet, Moody's, MT Newswires, Aiera, LSEG, PitchBook, Chronograph, Egnyte)

9. **Claude for Microsoft 365** — tooling แยกต่างหากสำหรับ deployment

10. **Making It Yours** — แนวทางการปรับแต่ง (swap connectors, add context, adjust scope)

11. **Skill & Command Reference** — รายละเอียดทั้ง 70+ skills รวมถึง commands ที่ใช้สำหรับแต่ละ vertical

## CLAUDE.md — การตั้งค่า Repository

`CLAUDE.md` เป็นเอกสารที่กำหนด **conventions** และ **workflow** สำหรับ repository:

### Convention ที่กำหนด

1. **Repository Structure** — ระบุวัฒนาการของ 4 ส่วนหลัก:
   - `plugins/agent-plugins/` — agents ชื่อที่ชัดเจน
   - `plugins/vertical-plugins/` — groups ของ skills, commands, MCPs
   - `plugins/partner-built/` — partner plugins (LSEG, S&P Global)
   - `managed-agent-cookbooks/` — manifests สำหรับ API deployment

2. **Key Files** — รายการไฟล์สำคัญ:
   - `marketplace.json` — registers all plugins
   - `plugin.json` — metadata ของแต่ละ plugin
   - `commands/*.md` — slash commands
   - `skills/*/SKILL.md` — knowledge base
   - `*.local.md` — user-specific config (gitignored)
   - `mcp-categories.json` — canonical MCP categories

3. **Development Workflow** — กระบวนการพัฒนา:
   - แก้ไข markdown ตรงๆ (changes take effect immediately)
   - ทดสอบ commands ด้วย `/plugin:command-name`
   - Skills ถูก invoke อัตโนมัติ

4. **Quality Assurance** — ตรวจสอบ:
   - ใช้ `python3 scripts/check.py` ก่อน commit
   - Lints ทุก manifest, verifies cross-file references
   - ตรวจสอบว่า bundled skills ไม่เบี่ยงเบน (drift) จากแหล่ง
   - **Edit skills ใน `vertical-plugins/` แล้วรัน `sync-agent-skills.py` เพื่อ propagate**

## LICENSE — Apache 2.0

Repository ใช้ **Apache License 2.0** — ประเภท permissive open-source license ที่อนุญาต:
- การใช้งานเชิงพาณิชย์
- การแก้ไข
- การแจกจ่าย
- การนำไปใช้เป็นส่วนประกอบขนาดใหญ่

**ข้อกำหนด**:
- ต้องอ้างอิงและบรรจุสำเนา license
- ต้องเปิดเผยการเปลี่ยนแปลง

**ไม่ครอบคลุม**:
- ลิขสิทธิ์เดิม ลิขสิทธิ์อื่นๆ, เครื่องหมายการค้า, สิทธิบัตร
- ความรับผิดชอบ — ไม่มีการรับประกัน

## .gitignore — ไฟล์ที่ไม่ติดตาม

`.gitignore` ป้องกันไฟล์ที่ไม่ควรอยู่ใน repository:

### ประเภท

| ประเภท | รูปแบบ | เหตุผล |
|---|---|---|
| **OS** | `.DS_Store`, `Thumbs.db` | ไฟล์ระบบ |
| **IDE** | `.vscode/`, `.idea/`, `*.swp`, `*~` | Editor artifacts |
| **Dependencies** | `node_modules/`, `vendor/`, `venv/`, `__pycache__/`, `*.pyc` | Third-party code |
| **Build** | `dist/`, `build/`, `out/`, `target/`, `*.log` | Output artifacts |
| **Secrets** | `.env`, `.env.local`, `.env.*.local`, `*.key`, `*.pem` | Credentials ต้องห้าม |
| **Testing** | `coverage/`, `.coverage`, `*.cover`, `.pytest_cache/` | Test artifacts |
| **Packages** | `*.egg-info/`, `.eggs/` | Python packages |
| **Personal** | `TASKS.md`, `MEMORY.md`, `.claude/worktrees/` | Local workflow files |

### ความสำคัญความปลอดภัย

- **ไม่เคยสำรอง `.env`, `*.key`, `*.pem`** — Gitleaks check (`.github/workflows/secret-scan.yml`) จะล้มหากมีการตรวจพบ
- `.env.local` ใช้สำหรับการตั้งค่าส่วนตัว (API keys, credentials)

---

**สรุป**: Root files กำหนด purpose (README), conventions (CLAUDE.md), legal terms (LICENSE), และความปลอดภัย (.gitignore) ของโครงการ
