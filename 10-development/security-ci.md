---
title: Security & CI
parent: 10 การ Development
nav_order: 4
---

# Security CI — Secret Scanning ใน GitHub Actions

## .github/workflows/secret-scan.yml

Repository มี GitHub Actions workflow ที่ run บน pull request + push ไปยัง main:

```yaml
name: secret-scan

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  gitleaks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
        with:
          fetch-depth: 0
      
      - name: gitleaks
        run: |
          set -euo pipefail
          curl -sSL -o gitleaks.tgz \
            https://github.com/gitleaks/gitleaks/releases/download/v8.28.0/gitleaks_8.28.0_linux_x64.tar.gz
          echo "a65b5253807a68ac0cafa4414031fd740aeb55f54fb7e55f386acb52e6a840eb  gitleaks.tgz" | sha256sum -c -
          tar -xzf gitleaks.tgz gitleaks
          ./gitleaks git --redact --exit-code 1 .
      
      - name: internal-reference scrub
        run: |
          set -euo pipefail
          if grep -rInE '\.ant\.dev|antspace\.dev|anthropic-internal|\bgo/[a-z][a-z0-9_-]+\b' \
              --include='*.md' --include='*.yaml' --include='*.yml' --include='*.json' \
              --include='*.py' --include='*.sh' \
              --exclude-dir=.github . ; then
            echo "::error::internal Anthropic references found above"
            exit 1
          fi
```

## Gitleaks v8.28.0 — Detect Secrets

Gitleaks scan git history หา patterns ของ secrets:

- AWS keys (`AKIA...`)
- API keys (bearer tokens, OAuth tokens)
- Private keys (RSA, SSH)
- Database passwords
- Stripe/Twilio/GitHub tokens

Run เมื่อ PR open หรือ push ไป main:

```bash
gitleaks git --redact --exit-code 1 .
```

Flags:

- `git` — scan git history (ไม่ just current files)
- `--redact` — redact secret values ใน output
- `--exit-code 1` — exit 1 ถ้า detect secrets (fail CI)

### ตัวอย่าง

File ใน PR มี:

```python
# ❌ BAD
api_key = "sk-ant-xxxxxxxxxxxxxxxxxxxxx"
```

Workflow:

```bash
gitleaks git ... .
# Findings:
# RuleID: anthropic-api-key
# File: example.py
# Match: sk-ant-xxxxxxxxxxxxxxxxxxxxx
```

Output:

```
::error::gitleaks found secrets
exit code: 1
```

PR check fail — merge ถูก block

## Internal Reference Scrub

สองหน้า: Anthropic internal references ที่ไม่ควรรั่ว:

```bash
grep -rInE '\.ant\.dev|antspace\.dev|anthropic-internal|\bgo/[a-z][a-z0-9_-]+\b' \
    --include='*.md' --include='*.yaml' --include='*.yml' --include='*.json' \
    --include='*.py' --include='*.sh' \
    --exclude-dir=.github . 
```

Patterns:

- `*.ant.dev` — internal Anthropic domains
- `antspace.dev` — internal platform
- `anthropic-internal` — literals
- `go/` shortcuts — Anthropic go-link system

ถ้า found:

```
::error::internal Anthropic references found above
exit code: 1
```

## Permissions Setup

Workflow ใช้ `permissions: contents: read`:

```yaml
permissions:
  contents: read    # ← read-only; can't push credentials
```

GitHub auto-revokes write/push tokens ถ้า job ไม่ request พวกมัน

## ไม่มี Test Workflow

Repository **ไม่มี** test CI (ไม่มี `*.test.yml`). Why?

- Scripts เป็น bash/Python utilities — ขึ้นอยู่กับ env (ANTHROPIC_API_KEY, MCP servers, ...)
- Manifests = data files (YAML, JSON) — linting cover โดย check.py
- Actual agent execution ต้อง Anthropic API → require API key ใน CI

Instead:

- `scripts/check.py` run locally ต่อก่อน commit
- `scripts/deploy-managed-agent.sh --dry-run` test manifest resolution
- Manual testing ใน Cowork/Claude Code ก่อนสั่ง PR

## MCP Credentials ใน Workflow

MCP connections (Daloopa, FactSet, Bloomberg, ...) ต้อง API keys

**ไม่ store ใน repo** — store ใน GitHub Secrets:

```bash
# GitHub repo settings → Secrets and variables → Actions

DALOOPA_API_KEY = sk-daloopa-xxxxx
FACTSET_API_KEY = sk-factset-xxxxx
LSEG_REFINITIV_KEY = sk-lseg-xxxxx
```

ใน workflow สามารถ access:

```yaml
- name: run tests
  env:
    DALOOPA_API_KEY: ${{ secrets.DALOOPA_API_KEY }}
  run: ./test-script.sh
```

GitHub mask secret ใน logs:

```
✓ Set environment variable DALOOPA_API_KEY=***
```

## Local Secret Check

ก่อน push locally:

```bash
# Install gitleaks locally
brew install gitleaks    # or download binary

# Scan
gitleaks detect --verbose --redact

# If found: remove + amend
git rm --cached filename
git commit --amend
```

## Commit Hook (Optional)

เพิ่ม pre-commit hook ใน `.git/hooks/pre-commit`:

```bash
#!/bin/bash
set -euo pipefail

# Install gitleaks if missing
if ! command -v gitleaks &>/dev/null; then
  echo "gitleaks not found — skipping secret scan"
  exit 0
fi

# Scan staged files
gitleaks protect --staged --exit-code 1
```

Make executable:

```bash
chmod +x .git/hooks/pre-commit
```

## Bypass Secret Check (Emergency Only)

ถ้า legitimate secrets ต้อง commit (เช่น test fixtures):

1. **Add ไป `.gitignore`** ก่อน (ให้ gitleaks skip)
2. **Document ใน README** ว่าทำไม
3. **Request allowlist exception** จากเพื่อน security reviewer

Example `.gitignore`:

```
# Test fixtures — contain fake credentials
examples/test-secrets.json
```

**Never** `--no-verify` — ทำให้ commit ข้าม hook

## Check Status

ดู workflow status:

```
GitHub repo → Actions → latest workflow run
```

ถ้า fail:

```
Artifacts → download logs
```

Log show:

- Gitleaks findings (if any)
- Internal references (if any)
- Exact file:line

---

**Next:** [Fork and customize — extend สำหรับ firm ของคุณ](fork-and-customize.md)
