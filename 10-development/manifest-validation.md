---
title: Manifest Validation
parent: 10 การ Development
nav_order: 1
---

# Manifest Validation — check.py

## check.py ทำอะไร

Script `scripts/check.py` lint ทุก plugin + managed-agent manifests — ตรวจว่า syntax valid, cross-references resolve, และ bundled skills ตรงกับ source

Run ก่อน `git push` ทุกครั้ง:

```bash
python3 scripts/check.py
```

Exit 0 = all clear, exit 1 = error found

## Checks ที่ทำ

### 1. YAML/JSON Syntax Parse

```bash
for *.yaml in managed-agent-cookbooks/
  yaml.safe_load()
for *.json in plugins/**/.claude-plugin/
  json.loads()
```

ถ้า syntax invalid:

```
YAML parse: managed-agent-cookbooks/gl-reconciler/agent.yaml: 
  while scanning a simple key in "...", could not find expected ':'
```

### 2. Plugin Manifest Parse

```bash
plugins/**/.claude-plugin/plugin.json        → valid JSON
.claude-plugin/marketplace.json              → valid JSON
managed-agent-cookbooks/*/steering-examples.json
```

Check required fields ใน plugin.json:

```json
{
  "name": "pitch-agent",
  "description": "...",
  "version": "0.1.0"
}
```

### 3. Agent Frontmatter Validation

แต่ละ agent markdown (`plugins/agent-plugins/<slug>/agents/<slug>.md`) ต้องมี valid YAML frontmatter:

```
---
name: Pitch Agent
description: "Generate branded pitch decks from comps, precedents, and LBO analysis"
---
```

Check:

```python
# Split frontmatter
_, fm, _ = text.split("---", 2)
meta = yaml.safe_load(fm)

# Require fields
for k in ("name", "description"):
  if k not in meta:
    error(f"missing '{k}'")
```

ถ้า missing:

```
frontmatter: plugins/agent-plugins/pitch-agent/agents/pitch-agent.md: 
  missing 'description'
```

### 4. Reference Resolution

Check ว่า `system.file`, `skills[].path`, `callable_agents[].manifest` ชี้ไปยังไฟล์ที่มีอยู่:

```yaml
# agent.yaml
system:
  file: ../../plugins/agent-plugins/gl-reconciler/agents/gl-reconciler.md
```

Check:

```python
p = (base / sys_spec["file"]).resolve()
if not p.is_file():
  error(f"system.file -> {file} (not found)")
```

ถ้า reference broken:

```
ref: managed-agent-cookbooks/gl-reconciler/agent.yaml: 
  system.file -> ../../agents/gl-reconciler.md (not found)
```

Skill paths:

```yaml
skills:
  - { path: ../../plugins/agent-plugins/gl-reconciler }
```

Callable agents:

```yaml
callable_agents:
  - manifest: ./subagents/reader.yaml
```

### 5. Bundled Skill Drift Detection

Agent plugins bundle skills จาก verticals — ต้องตรงกับ source:

```
plugins/agent-plugins/pitch-agent/skills/comps-analysis/
  ↑ should match ↓
plugins/vertical-plugins/investment-banking/skills/comps-analysis/
```

Check ใช้ `filecmp.dircmp()`:

```python
src = src_by_name.get(bundled.name)
cmp = filecmp.dircmp(src, bundled)
if cmp.diff_files or cmp.left_only or cmp.right_only:
  error(f"bundled-skill drifted — run scripts/sync-agent-skills.py")
```

ถ้า drift:

```
bundled-skill: plugins/agent-plugins/pitch-agent/skills/comps-analysis/: 
  drifted from plugins/vertical-plugins/investment-banking/skills/comps-analysis/
  (run scripts/sync-agent-skills.py)
```

### 6. Agent Prose Skill References

หากนาย `pitch-agent.md` system prompt reference skill ที่ไม่มีใน bundled skills — error:

```markdown
---
name: Pitch Agent
---

Recommend using the `comps-analysis` skill to find peer trading multiples.
```

Check ว่า `comps-analysis` มีอยู่ใน `plugins/agent-plugins/pitch-agent/skills/`:

```python
for ref in set(re.findall(r"`([a-z0-9]+(?:-[a-z0-9]+)+)`", md.read_text())):
  if ref in src_by_name and ref not in bundle:
    error(f"references `{ref}` but not bundled")
```

Error:

```
agent-prose: plugins/agent-plugins/pitch-agent/agents/pitch-agent.md: 
  references `comps-analysis` but 
  plugins/agent-plugins/pitch-agent/skills/comps-analysis/ is not bundled
```

### 7. Marketplace Source Paths

`.claude-plugin/marketplace.json` ต้องมี valid source paths:

```json
{
  "plugins": [
    {
      "name": "pitch-agent",
      "source": "./plugins/agent-plugins/pitch-agent"
    }
  ]
}
```

Check:

```python
src = (ROOT / p["source"]).resolve()
if not (src / ".claude-plugin" / "plugin.json").is_file():
  error(f"source -> {p['source']} (no plugin.json)")
```

### 8. Required Files Per Managed Agent

แต่ละ managed-agent ต้องมี:

```
managed-agent-cookbooks/gl-reconciler/
├── agent.yaml               (required)
├── README.md                (required)
├── steering-examples.json   (required)
└── subagents/               (if callable_agents in agent.yaml)
```

Check:

```python
for d in MANAGED.iterdir():
  for req in ("agent.yaml", "README.md", "steering-examples.json"):
    if not (d / req).is_file():
      error(f"missing: {d.name}/{req}")
```

## ใช้ check.py ใน CI/CD

GitHub Actions workflow `.github/workflows/secret-scan.yml` ไม่ run `check.py` แต่คุณ add ได้:

```yaml
- name: manifest validation
  run: |
    python3 -m pip install pyyaml
    python3 scripts/check.py
```

## Error Resolution

### "YAML parse" error

Fix syntax error ใน .yaml file — use online YAML validator

### "Reference not found" error

Verify path relative ต่อ manifest directory:

```yaml
# ถูก (relative to agent.yaml parent)
system:
  file: ../../plugins/agent-plugins/gl-reconciler/agents/gl-reconciler.md

# ผิด (absolute path, ใช้ไม่ได้)
system:
  file: /home/user/repo/plugins/...
```

### "Bundled skill drift"

Run sync script:

```bash
python3 scripts/sync-agent-skills.py
git add plugins/agent-plugins/*/skills/
git commit -m "Sync bundled skills from verticals"
```

### "Missing required file"

Create the file:

```bash
touch managed-agent-cookbooks/my-agent/README.md
echo '# My Agent' > managed-agent-cookbooks/my-agent/README.md
```

---

**Next:** [Skill sync — keep bundles in sync with sources](skill-sync.md)
