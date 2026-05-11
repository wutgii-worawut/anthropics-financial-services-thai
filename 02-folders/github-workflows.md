---
title: GitHub Workflows และ Secret Scanning
parent: 02 โครงสร้างโฟลเดอร์
nav_order: 3
---

# GitHub Workflows Configuration

ที่อยู่: `.github/workflows/`

Repository นี้ใช้ **GitHub Actions** เพื่อการความปลอดภัยอัตโนมัติ — ตรวจสอบความลับก่อนที่จะ merge code

## secret-scan.yml — รายละเอียด

```yaml
name: secret-scan
on:
  pull_request:
  push:
    branches: [main]
permissions:
  contents: read
```

### Triggers

Workflow ทำงาน:
1. **ทุก Pull Request** — เมื่อมี PR ใหม่หรือ update
2. **ทุก Push to main** — เมื่อ commit ผ่ายไปยัง main branch

### Job 1: Gitleaks Integration

```bash
curl -sSL -o gitleaks.tgz \
  https://github.com/gitleaks/gitleaks/releases/download/v8.28.0/gitleaks_8.28.0_linux_x64.tar.gz
echo "a65b5253807a68ac0cafa4414031fd740aeb55f54fb7e55f386acb52e6a840eb  gitleaks.tgz" | sha256sum -c -
tar -xzf gitleaks.tgz gitleaks
./gitleaks git --redact --exit-code 1 .
```

**Gitleaks** คืออะไร:
- Tool ที่ค้นหา credentials, API keys, tokens, private keys ในเนื้อหา git
- Scans: commit history, file contents, tags

**ไหลของการทำงาน**:
1. ดาวน์โหลด gitleaks binary (v8.28.0)
2. ตรวจสอบ SHA256 checksum (security: ตรวจสอบไฟล์จริง)
3. Extract binary จาก tarball
4. รัน: `gitleaks git --redact --exit-code 1 .`
   - `git` = scan git history
   - `--redact` = hide actual secrets ในผลลัพธ์
   - `--exit-code 1` = fail (exit 1) ถ้าพบความลับ

**ความลับที่จะตรวจพบ** (default patterns):
- AWS keys (`AKIA...`)
- GitHub tokens (`ghp_...`, `ghs_...`)
- Private keys (`-----BEGIN RSA PRIVATE KEY-----`)
- Slack/Discord tokens
- Database passwords
- API keys (Stripe, SendGrid, etc.)

### Job 2: Internal Reference Scrub

```bash
if grep -rInE '\.ant\.dev|antspace\.dev|anthropic-internal|\bgo/[a-z][a-z0-9_-]+\b' \
    --include='*.md' --include='*.yaml' --include='*.yml' --include='*.json' \
    --include='*.py' --include='*.sh' \
    --exclude-dir=.github . ; then
  echo "::error::internal Anthropic references found above"
  exit 1
fi
```

**วัตถุประสงค์**: ลบการอ้างอิงภายใน Anthropic (ไม่สามารถแบ่งปันแบบสาธารณะได้)

**ตัวอักษรที่ค้นหา** (regex):
- `.ant.dev` — internal Anthropic domain
- `antspace.dev` — internal domain
- `anthropic-internal` — literal string
- `go/[a-z]...` — go/ short links (internal only)

**ไฟล์ที่ค้นหา**: 
- `*.md`, `*.yaml`, `*.yml`, `*.json`, `*.py`, `*.sh`

**ไฟล์ที่ข้าม**:
- `.github/` directory (workflow files themselves อาจมีการอ้างอิงเพื่อเอกสาร)

**ถ้าพบ**: exit 1 → **workflow fails** → **PR cannot merge**

## เหตุที่ไม่มี Build/Test Workflows

Repository นี้ **มี _เพียง_ secret-scan workflow เท่านั้น** — ไม่มี CI/CD สำหรับ testing, linting, หรือ build

### เหตุผล

1. **Repository เป็น Markdown/JSON/YAML** — ไม่มี executable code ที่ต้อง compile
   
2. **ไม่มี build artifact** — plugins คือ configuration files เท่านั้น
   
3. **Local validation เพียงพอ**:
   ```bash
   # Developer รัน locally ก่อน push:
   python3 scripts/check.py          # Lint manifests
   python3 scripts/sync-agent-skills.py  # Sync bundled skills
   bash scripts/test-cookbooks.sh    # Dry-run managed-agent cookbooks
   ```

4. **Cowork/API validates at runtime** — plugins ทำงานจริงผ่าน Claude platform ซึ่งแล้ว validate ตัวเอง

5. **Secret scanning มีความสำคัญมากกว่า** — financial services code ต้องป้องกันผู้ใช้จากการจำหน่าย secrets (API keys, credentials)

## Workflow Execution Flow

```
┌─────────────────────────────────────────────────────┐
│ Developer: git push / git push PR                   │
└─────────────┬───────────────────────────────────────┘
              │
              ↓
┌─────────────────────────────────────────────────────┐
│ GitHub detects: push to main OR PR open            │
│ Triggers: secret-scan.yml                          │
└─────────────┬───────────────────────────────────────┘
              │
              ↓ (parallel jobs)
    ┌─────────────────────────┬─────────────────────────┐
    │ Job 1: Gitleaks         │ Job 2: Internal Ref     │
    │ ✓ No credentials found  │ ✓ No internal refs      │
    │   OR                    │   OR                    │
    │ ✗ Fail (exit 1)         │ ✗ Fail (exit 1)        │
    └─────────────┬───────────┴──────────────┬──────────┘
                  │                          │
    ┌─────────────▼──────────────────────────▼──────────┐
    │ Both pass? → Green check ✓                         │
    │ Any fails? → Red X ✗ (PR cannot merge)            │
    └────────────────────────────────────────────────────┘
```

## Best Practices Enforced

| Enforcement | Reason |
|---|---|
| No credentials in files | Prevent accidental leaks |
| No internal references | Keep code shareable |
| Checked on every PR + push | Early detection |
| Exit code 1 = fail | Prevents merging unsafe code |
| SHA256 checksum on gitleaks binary | Ensure binary integrity |
| `--redact` on output | Don't echo secrets in logs |

## Related Security

### .gitignore (defense 1)

Prevents committing secrets in first place:
```
.env
.env.local
*.key
*.pem
```

### secret-scan.yml (defense 2)

Catches any secrets that slip past `.gitignore`

### CLAUDE.md note

```
- ไม่ commit secrets — `.env`, credentials, API keys, OAuth tokens, private keys, passwords
```

---

**สรุป**: secret-scan workflow ป้องกัน repository จากการจำหน่าย credentials และการอ้างอิงภายในอัตโนมัติ — ทำงาน ทุก PR + push โดยใช้ Gitleaks + regex checks
