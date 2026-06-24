# DevSecOps for Git

> **"Your Git repository is your most critical security boundary — it's where source code, secrets, and deployment instructions live. Protect it first."**

This guide explains how to implement **Shift Left security practices within Git workflows**. From pre-commit hooks that block secrets to branch protection policies that enforce review, every Git operation is an opportunity to prevent a vulnerability.

---

## Table of Contents

- [Why Git Security Is DevSecOps](#why-git-security-is-devsecops)
- [The Git Threat Landscape](#the-git-threat-landscape)
- [Shift Left in Git: The 4 Gates](#shift-left-in-git-the-4-gates)
- [Pre-Commit Hooks (Local)](#pre-commit-hooks-local)
- [Gitleaks Setup: Local + GitHub Actions](#gitleaks-setup-local--github-actions)
- [Pre-Commit Hooks Deep Dive](#pre-commit-hooks-deep-dive)
- [.gitignore Best Practices](#gitignore-best-practices)
- [Branch Protection (Remote)](#branch-protection-remote)
- [CI/CD Pipeline Security (Automation)](#cicd-pipeline-security-automation)
- [Commit Signing & Verification](#commit-signing--verification)
- [GitOps Security](#gitops-security)
- [Secrets Management in Git](#secrets-management-in-git)
- [Supply Chain Security via Git](#supply-chain-security-via-git)
- [Git Security Tooling](#git-security-tooling)
- [Quick Reference: Secure Git Workflow](#quick-reference-secure-git-workflow)
- [Summary](#summary)

---

## Why Git Security Is DevSecOps

Git is the **single source of truth** for modern software delivery. It contains:

- **Source code** — your application's logic
- **Secrets** — API keys, tokens, credentials (often accidentally)
- **Configuration** — IaC, CI/CD pipelines, environment settings
- **Dependencies** — `package.json`, `requirements.txt`, `go.mod`
- **Deployment instructions** — Dockerfiles, Kubernetes manifests, Helm charts

If Git is compromised, **everything downstream is compromised**.

| Git Asset | Downstream Impact if Compromised |
|---|---|
| Source code | Backdoored application deployed to production |
| CI/CD config | Malicious pipeline injects malware during build |
| IaC (Terraform) | Attacker provisions cryptocurrency miners in your cloud |
| Dependencies | Supply chain attack poisons all consumers |
| Signed release tags | Forged update shipped to all users |

> **In DevSecOps, "shift left" starts at `git init`.**

---

## The Git Threat Landscape

### What attackers target in Git

| Attack | How It Works | Business Impact |
|---|---|---|
| **Secrets in history** | Developer commits `.env` file with AWS key | Attacker exfiltrates data, spins up EC2 instances |
| **Force push to main** | Attacker overwrites `main` with malicious code | Production deployed with backdoor |
| **Branch hijacking** | `feature-backdoor` merged via PR without review | Malicious dependency injected |
| **Supply chain via git** | Attacker compromises GitHub account, pushes malicious tag | All users update to backdoored version |
| **Submodule poisoning** | Malicious submodule URL redirects to attacker repo | Build process executes attacker-controlled code |
| **Merge conflict exploitation** | Attacker resolves conflict by re-introducing vulnerability | Old bug returns to production |
| **Rebase rewriting** | Malicious rebase quietly removes security controls | Security hooks, checks removed from history |
| **Git LFS abuse** | Large file stored with malware | Build server downloads and executes |

---

## Shift Left in Git: The 4 Gates

DevSecOps in Git operates across **4 gates** where security controls are enforced:

```
Developer → Pre-commit → Push → Pull Request → Merge → CI/CD → Production
     │          │          │          │          │         │
     ▼          ▼          ▼          ▼          ▼         ▼
  [Create]   [Verify]   [Scan]   [Review]   [Build]   [Deploy]
   Local      Local     Remote    Remote   Automation  Verified
```

| Gate | What It Blocks | Security Control |
|---|---|---|
| **1. Pre-commit (Local)** | Secrets, bad patterns | Git hooks, linting, SAST |
| **2. Push (Remote)** | Unverified code | Branch protection, required reviews |
| **3. PR / Merge (Remote)** | Malicious changes | Code review, automated checks, DCO sign-off |
| **4. CI/CD (Automation)** | Vulnerable builds | SAST, SCA, container scan, deployment signing |

If an attacker bypasses one gate, the next gate catches them.

---

## Pre-Commit Hooks (Local)

Pre-commit hooks run **before a commit is created**. They are the fastest, cheapest security gate.

### How Pre-Commit Works

```
Developer runs: git commit -m "add login feature"
    │
    ▼
[pre-commit hook] ► runs security checks
    │
    ├── ❌ FAIL ► commit BLOCKED, developer must fix
    │
    └── ✅ PASS ► commit proceeds
```

### Essential Pre-Commit Security Hooks

```yaml
# .pre-commit-config.yaml
repos:
  # 1. Secrets Detection
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.2
    hooks:
      - id: gitleaks

  # 2. Python Security
  - repo: https://github.com/PyCQA/bandit
    rev: 1.7.7
    hooks:
      - id: bandit
        args: ["-c", "bandit.yaml"]

  # 3. General File Checks
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: check-added-large-files   # Prevents giant files
      - id: check-case-conflict       # Prevents case-sensitivity issues
      - id: detect-aws-credentials    # Catches AWS keys

  # 4. IaC Security
  - repo: https://github.com/bridgecrewio/checkov
    rev: 3.1.0
    hooks:
      - id: checkov
        files: \.(tf|json|yaml)$

  # 5. Container Security
  - repo: https://github.com/hadolint/hadolint
    rev: v2.12.0
    hooks:
      - id: hadolint-docker

  # 6. Commit Message Validation
  - repo: https://github.com/commitizen-tools/commitizen
    rev: v3.13.0
    hooks:
      - id: commitizen
        stages: [commit-msg]
```

### Installing Pre-Commit Locally

```bash
# 1. Install pre-commit framework
pip install pre-commit

# 2. Install hooks into your Git repo
cd my-project
git init
pre-commit install
pre-commit install --hook-type commit-msg

# 3. Test: try to commit a secret
echo "AWS_SECRET_ACCESS_KEY=AKIAIOSFODNN7EXAMPLE" > bad_file.txt
git add bad_file.txt
git commit -m "test secret"

# Expected output:
# Detect hardcoded secrets...............................Failed
# - hook id: gitleaks
# - exit code: 1
#
# 	○
# 	│╲
# 	│ ○
# 	○ ░
# 	░   gitleaks
#
# 9:16AM INF 1 leaks detected. 1 commits scanned
```

### Bypassing Pre-Commit Is an Anti-Pattern

```bash
# ❌ NEVER do this in production teams
GIT_LFS_SKIP_SMUDGE=1 git commit -m "skip hooks" --no-verify

# ✅ Instead: if a check is noisy, TUNE the rule, don't bypass it
```

**Policy:** Engineers who push with `--no-verify` without security team approval should have commits automatically flagged for audit.

---

## Gitleaks Setup: Local + GitHub Actions

Gitleaks is the most popular open-source tool for **detecting hardcoded secrets** in Git repositories. It scans commits, branches, pull requests, and pull request diffs for over 170+ secret types including AWS keys, GitHub tokens, Slack webhooks, database connection strings, and private keys.

### Architecture

```
Developer machine                      GitHub Cloud
┌─────────────────┐                  ┌─────────────────┐
│ git add .       │                  │ PR opened       │
│ git commit -m   │ ──► gitleaks     │                 │ ──► gitleaks
│                 │    protect       │                 │    scan
│                 │    (pre-commit)  │                 │    (CI)
└─────────────────┘                  └─────────────────┘
        │                                    │
        ▼                                    ▼
   Commit BLOCKED                      CI check FAILS
   if secret found                     PR cannot merge
```

---

### Part 1: Local Machine Setup

#### Step 1: Install Gitleaks

**macOS (Homebrew):**
```bash
brew install gitleaks
```

**Linux (Debian/Ubuntu):**
```bash
# Download latest release
wget https://github.com/gitleaks/gitleaks/releases/download/v8.18.2/gitleaks_8.18.2_linux_x64.tar.gz
tar -xzf gitleaks_8.18.2_linux_x64.tar.gz
sudo mv gitleaks /usr/local/bin/
gitleaks version
```

**Linux (Alpine/Docker):**
```bash
# Gitleaks is available as a Docker image
docker pull zricethezav/gitleaks:latest
```

**Windows (Chocolatey):**
```powershell
choco install gitleaks
```

**Windows (Scoop):**
```powershell
scoop install gitleaks
```

#### Step 2: Verify Installation

```bash
gitleaks version
# Output: 8.18.2
```

#### Step 3: Scan Your Repository

```bash
# Navigate to your project
cd my-project

# Scan your entire repository history
gitleaks detect --source . --verbose

# Scan only the latest commit
gitleaks detect --source . --no-git

# Scan specific commit range
gitleaks detect --source . --log-opts="--all --full-history" --verbose

# Scan a single branch
gitleaks detect --source . --log-opts="--all --full-history main"
```

#### Step 4: Interpret Results

```bash
$ gitleaks detect --source . --verbose

    ○
    │╲
    │ ○
    ○ ░
    ░   gitleaks

9:30AM INF 3 commits scanned.
9:30AM INF scan completed in 1.2s
9:30AM WRN leaks found: 2

{
  "Description": "AWS Access Key",
  "StartLine": 12,
  "EndLine": 12,
  "StartColumn": 24,
  "EndColumn": 44,
  "Match": "AKIAIOSFODNN7EXAMPLE",
  "Secret": "AKIAIOSFODNN7EXAMPLE",
  "File": "config/.env",
  "Commit": "abc123def",
  "Entropy": 3.8,
  "Author": "Devon Developer",
  "Email": "devon@company.com",
  "Date": "2026-06-20T09:15:00Z",
  "Message": "add AWS config",
  "Tags": ["aws", "key"]
}
```

#### Step 5: Set Up Pre-Commit Hook (Recommended)

Gitleaks integrates with the **pre-commit** framework for automatic scanning before every commit.

**Install pre-commit:**
```bash
pip install pre-commit
```

**Create `.pre-commit-config.yaml` in your repo:**
```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.2
    hooks:
      - id: gitleaks
        name: Detect hardcoded secrets
        description: Scans commits for secrets using gitleaks
        entry: gitleaks protect --verbose --redact --staged
        language: system
        pass_filenames: false
```

**Install the hook:**
```bash
pre-commit install
pre-commit install --hook-type pre-push
```

**Test it:**
```bash
# Create a test file with a fake secret
echo "AWS_SECRET_ACCESS_KEY=AKIAIOSFODNN7EXAMPLE" > test-secret.txt
git add test-secret.txt
git commit -m "test gitleaks"

# Expected output:
# Detect hardcoded secrets...............................Failed
# - hook id: gitleaks
# - exit code: 1
#
#     ○
#     │╲
#     │ ○
#     ○ ░
#     ░   gitleaks
#
# 9:35AM INF 1 leaks detected. 1 commits scanned
```

#### Step 6: Clean Up Test File

```bash
git checkout -- test-secret.txt
rm test-secret.txt
```

---

### Part 2: GitHub Actions Setup

Gitleaks runs in GitHub Actions to **scan pull requests, commits, and full history** before code is merged.

#### Option A: Official Gitleaks GitHub Action (Recommended)

```yaml
# .github/workflows/gitleaks.yml
name: Gitleaks Secret Scan

on:
  push:
    branches: [main, develop, release/*]
  pull_request:
    branches: [main, develop]
  workflow_dispatch:

jobs:
  gitleaks-scan:
    name: Scan for Secrets
    runs-on: ubuntu-latest
    steps:
      # 1. Checkout code (with full history for complete scan)
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0        # Full git history

      # 2. Run Gitleaks
      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}  # For private repos
```

#### Option B: Manual Gitleaks in GitHub Actions (More Control)

```yaml
# .github/workflows/gitleaks-custom.yml
name: Gitleaks Custom Scan

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]
  workflow_dispatch:

jobs:
  gitleaks:
    name: Secret Detection
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Install Gitleaks
        run: |
          wget https://github.com/gitleaks/gitleaks/releases/download/v8.18.2/gitleaks_8.18.2_linux_x64.tar.gz
          tar -xzf gitleaks_8.18.2_linux_x64.tar.gz
          sudo mv gitleaks /usr/local/bin/

      - name: Scan Repository
        run: |
          gitleaks detect \
            --source . \
            --verbose \
            --redact \
            --report-format json \
            --report-path gitleaks-report.json

      - name: Upload Report
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: gitleaks-report
          path: gitleaks-report.json

      - name: Comment PR
        if: github.event_name == 'pull_request' && failure()
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '🔴 **Gitleaks found secrets in this PR.**\n\nPlease remove hardcoded credentials and use environment variables or a secret manager instead.\n\nSee the uploaded artifact for details.'
            });
```

#### Option C: Scan Only PR Changes (Fastest)

```yaml
# .github/workflows/gitleaks-pr.yml
name: Gitleaks PR Scan

on:
  pull_request:
    branches: [main]

jobs:
  gitleaks-pr:
    name: Scan PR for Secrets
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run Gitleaks on PR
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          config-path: .gitleaks.toml
```

---

### Part 3: Gitleaks Configuration

By default, Gitleaks detects 170+ secret types. You can customize behavior with `.gitleaks.toml` or `.gitleaksignore`.

#### Basic Configuration File

```toml
# .gitleaks.toml
title = "My Project Gitleaks Config"

# ─── Allowlist: Files and paths to ignore ──────────────────
[allowlist]
paths = [
  '''\.env\.example$''',
  '''\.env\.local$''',
  '''\.env\.test$''',
  '''test\/fixtures\/''',
  '''test\/data\/''',
  '''\.terraform\/''',
  '''node_modules\/''',
  '''vendor\/''',
]

# ─── Allowlist: Specific regex matches ─────────────────────
regexes = [
  # Ignore example/test keys (fake patterns)
  '''EXAMPLE''',
  '''FAKE''',
  '''TEST''',
  '''DEMO''',
  '''MOCK''',
  # Ignore placeholder values
  '''your-key-here''',
  '''<API_KEY>''',
]

# ─── Allowlist: Specific commits ────────────────────────────
commits = [
  "abc123def456789",
]

# ─── Allowlist: Entropy exceptions ─────────────────────────
# High entropy false positives in test data
[allowlist.entropy]
Min = "3.0"
Max = "7.0"
```

#### Ignore Specific Findings

If you have a false positive, create `.gitleaksignore`:

```
# .gitleaksignore - one fingerprint per line
# Format: commit:file:startline:endline:match

# Example: test fixture with fake data
abc123def456789:tests/fixtures/aws_credentials.json:5:25:AKIAIOSFODNN7EXAMPLE

# Another example: documentation example key
f23456789012345:docs/setup.md:12:44:ghp_xxxxxxxxxxxxxxxxxxxx
```

#### Generate Ignore File from Existing Findings

```bash
# First, run gitleaks and generate report
gitleaks detect --source . --report-format json --report-path gitleaks.json

# Create .gitleaksignore from the report
# (extract fingerprints and add to the file)
cat gitleaks.json | jq -r '.[] | .Fingerprint' >> .gitleaksignore
```

**Important:** Never add *real* secrets to `.gitleaksignore` — only test fixtures, documentation examples, and false positives. If a real secret is found, **rotate it, don't ignore it.**

---

### Part 4: What Gitleaks Detects

Gitleaks detects 170+ secret types out of the box, including:

| Secret Type | Example Match | Risk |
|---|---|---|
| **AWS Access Key** | `AKIAIOSFODNN7EXAMPLE` | Full AWS account access |
| **AWS Secret Key** | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` | Full AWS account access |
| **GitHub Personal Token** | `ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` | Repository/code access |
| **GitHub OAuth App** | `gho_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` | Account takeover |
| **Slack Webhook** | `https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXX` | Spam/spy on channel |
| **Stripe API Key** | `sk_live_EXAMPLE_DO_NOT_USE` | Financial access |
| **SendGrid API Key** | `SG.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` | Email abuse |
| **Private Key (PEM)** | `-----BEGIN RSA PRIVATE KEY-----` | Decryption, impersonation |
| **Database URL** | `postgres://user:password@host:5432/db` | Database compromise |
| **JWT Token** | `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...` | Authentication bypass |
| **Discord Token** | `EXAMPLE_DISCORD_TOKEN_NOT_REAL` | Bot impersonation |
| **Twilio API Key** | `SKxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` | SMS abuse |
| **npm Token** | `npm_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` | Package registry access |
| **SSH Private Key** | `-----BEGIN OPENSSH PRIVATE KEY-----` | Server access |
| **Dynatrace Token** | `dt0c01.xxxxxxx.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` | Monitoring data access |

Full list: [Gitleaks Default Rules](https://github.com/gitleaks/gitleaks#rules-summary)

---

### Part 5: Recovery Workflow

If Gitleaks finds a real secret in history:

```bash
# Step 1: IMMEDIATELY rotate the secret
aws iam update-access-key --access-key-id AKIAIOSFODNN7EXAMPLE \
    --status Inactive --user-name dev-user

# Step 2: Confirm no unauthorized access
# Check AWS CloudTrail, GitHub audit logs, etc.

# Step 3: Remove secret from history (requires force push!)
# Use BFG Repo-Cleaner (fast, easy)
# Or git-filter-repo (modern replacement for git filter-branch)

git filter-repo --path config/.env --invert-paths
# OR: BFG
docker run --rm -it -v $(pwd):/repo \
  bfg-repo-cleaner:latest \
  --delete-files .env /repo

# Step 4: Verify secret is gone
git log --all --full-history --grep="AKIAIOSFODNN7EXAMPLE"

# Step 5: Force push (coordinate with team!)
git push --force-with-lease origin main

# Step 6: Verify Gitleaks passes
gitleaks detect --source . --verbose

# Step 7: Update pre-commit to prevent recurrence
pre-commit run --all-files

# Step 8: Audit all forks (they still have the old history!)
# Notify fork owners to rebase or delete
```

**⚠️ Critical:** Rewriting history breaks forks, open PRs, and local clones. Coordinate with the team before force-pushing.

---

### Part 6: Advanced Patterns

#### Docker-Based Scanning

```bash
# Use gitleaks via Docker (no local install)
docker run --rm -v $(pwd):/code \
  zricethezav/gitleaks:latest detect \
  --source /code --verbose
```

#### CI Scan with Baseline

For large legacy repos, establish a baseline and only alert on new findings:

```bash
# Generate baseline (existing secrets you can't immediately fix)
gitleaks detect --source . --report-format json \
  --report-path gitleaks-baseline.json

# In CI: scan but allowlist existing findings
gitleaks detect --source . \
  --baseline-path gitleaks-baseline.json \
  --verbose
```

#### Multi-Repo Scanning

```bash
# Scan multiple repos in a monorepo
#!/bin/bash
REPOS=("frontend" "backend" "infrastructure")

for repo in "${REPOS[@]}"; do
  echo "Scanning $repo..."
  gitleaks detect --source "$repo" --report-format json \
    --report-path "gitleaks-$repo.json"
done
```

---

### Quick Reference: Gitleaks Commands

| Command | Purpose |
|---|---|
| `gitleaks version` | Check version |
| `gitleaks detect --source .` | Scan current directory |
| `gitleaks detect --source . --verbose` | Verbose output |
| `gitleaks detect --source . --no-git` | Scan files only (no history) |
| `gitleaks detect --source . --redact` | Redact secrets in output |
| `gitleaks protect --staged` | Scan staged files only (pre-commit) |
| `gitleaks detect --source . --config .gitleaks.toml` | Custom config |
| `gitleaks detect --source . --report-format json --report-path report.json` | JSON report |
| `gitleaks detect --source . --baseline-path baseline.json` | Baseline scan |
| `gitleaks detect --source . --log-opts="--since=2025-01-01"` | Date-filtered scan |

---

### Summary

| Layer | Setup | When It Runs |
|---|---|---|
| **Local** | `pre-commit install` + Gitleaks hook | Before every commit |
| **Push** | GitHub branch protection | Prevents push of unverified commits |
| **PR** | GitHub Actions `gitleaks-action` | On every pull request |
| **Full History** | CI scheduled scan | Nightly or weekly scan of all history |
| **Remediation** | `.gitleaks.toml` + `.gitleaksignore` | Configure and manage findings |

> **Gitleaks is the first gate in your DevSecOps Git pipeline. Install it locally today, add it to CI tomorrow, and sleep better tonight.**

---

## Pre-Commit Hooks Deep Dive

> **"A pre-commit hook is the fastest, cheapest security control that exists. It runs in under a second, costs zero dollars, and prevents secrets from ever entering your repository."**

### What Is a Pre-Commit Hook?

A **pre-commit hook** is a script that Git executes **automatically before a commit is created**. If the script exits with a non-zero code, the commit is **blocked**.

```
Developer types: git commit -m "add feature"
    │
    ├─► Git stages files in .git/index
    │
    ▼
┌──────────────────┐
│  pre-commit hook │  ◄── Your script runs HERE
│  (executable)    │
└──────────────────┘
    │
    ├─❌ FAIL (exit code ≠ 0) ──► Commit BLOCKED
    │
    └─✅ PASS (exit code 0) ────► Commit created
```

### Where Hooks Live

```
my-project/
├── .git/
│   └── hooks/              ◄── Git hooks live here
│       ├── applypatch-msg.sample
│       ├── commit-msg.sample
│       ├── pre-commit        ◄── Runs before commit
│       ├── pre-commit.sample
│       ├── pre-push.sample
│       ├── pre-rebase.sample
│       └── ...
├── src/
├── package.json
└── README.md
```

### The Complete Hook Lifecycle

Git supports hooks at **8 lifecycle stages**:

| Hook | When It Fires | Common Use |
|---|---|---|
| `pre-commit` | Before commit is created | Secrets scan, lint, format |
| `prepare-commit-msg` | Before message editor opens | Auto-add ticket ID |
| `commit-msg` | After message is entered | Enforce commit format |
| `post-commit` | After commit is created | Notify, update ticket |
| `pre-push` | Before `git push` | Full tests, build check |
| `pre-rebase` | Before `git rebase` | Warn about shared branches |
| `post-merge` | After `git merge` | Reinstall deps |
| `post-checkout` | After `git checkout` | Fix permissions, hooks install |

Visual flow:

```
Developer                              Git Server
    │                                      │
    ├── git add . ──► Index staged         │
    │                                      │
    ├── git commit ──► pre-commit ────────►│
    │   (BLOCKED if ❌)                    │
    │                                      │
    ├── git push ──► pre-push ────────────►│──► pre-receive (server)
    │   (BLOCKED if ❌)                    │    (BLOCKED if ❌)
    │                                      │
    └── Code merged ───────────────────────► deploys to CI/CD
```

### Writing a Raw Pre-Commit Hook

```bash
#!/bin/sh
# .git/hooks/pre-commit  (must be executable: chmod +x)

# Color output for readability
RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m' # No Color

echo "[pre-commit] Running security checks..."

# ─── 1. Detect Secrets ──────────────────────
gitleaks protect --staged --verbose --redact
if [ $? -ne 0 ]; then
    echo "${RED}❌ Gitleaks found secrets!${NC}"
    echo "   Remove hardcoded credentials or update .gitleaks.toml allowlist."
    exit 1
fi

# ─── 2. Check for Private Keys ──────────────
if git diff --cached --name-only | xargs grep -l "BEGIN OPENSSH PRIVATE KEY" 2>/dev/null; then
    echo "${RED}❌ Private key committed!${NC}"
    exit 1
fi

# ─── 3. Run Linter (example: ESLint) ────────
npm run lint:staged
if [ $? -ne 0 ]; then
    echo "${RED}❌ Lint errors!${NC} Fix before committing."
    exit 1
fi

# ─── 4. Run Unit Tests (fast subset) ────────
npm test -- --testPathPattern="unit" --bail
if [ $? -ne 0 ]; then
    echo "${RED}❌ Tests failed!${NC} Fix before committing."
    exit 1
fi

echo "${GREEN}✅ All pre-commit checks passed.${NC}"
exit 0
```

Enable it:
```bash
chmod +x .git/hooks/pre-commit
```

### Using the Pre-Commit Framework (Recommended)

Managing hooks by hand is fragile. The **pre-commit** framework centralizes and shares hooks.

**Install:**
```bash
pip install pre-commit
```

**Create `.pre-commit-config.yaml`:**
```yaml
repos:
  # Security: Secrets Detection
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.2
    hooks:
      - id: gitleaks
        name: Detect secrets in staged files

  # Security: General file checks
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: check-added-large-files
        args: ['--maxkb=1000']
      - id: check-case-conflict
      - id: detect-private-key
      - id: detect-aws-credentials
      - id: forbid-new-submodules

  # Security: Dockerfile linting
  - repo: https://github.com/hadolint/hadolint
    rev: v2.12.0
    hooks:
      - id: hadolint

  # Security: IaC scanning
  - repo: https://github.com/bridgecrewio/checkov
    rev: 3.2.0
    hooks:
      - id: checkov
        files: \.(tf|json|yaml)$

  # Code Quality: Python
  - repo: https://github.com/PyCQA/bandit
    rev: 1.7.7
    hooks:
      - id: bandit
        args: ["-c", "bandit.yaml"]

  # Code Quality: Linting
  - repo: https://github.com/pre-commit/mirrors-eslint
    rev: v9.0.0
    hooks:
      - id: eslint

  # Commit Message Formatting
  - repo: https://github.com/commitizen-tools/commitizen
    rev: v3.13.0
    hooks:
      - id: commitizen
        stages: [commit-msg]
```

**Install hooks into repo:**
```bash
cd my-project
pre-commit install
pre-commit install --hook-type commit-msg
```

**Run on all files (first-time setup):**
```bash
pre-commit run --all-files
```

### Common Pre-Commit Hook Configurations

#### For Node.js Projects

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.2
    hooks:
      - id: gitleaks

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: check-added-large-files
      - id: detect-private-key

  - repo: local
    hooks:
      - id: eslint
        name: ESLint
        entry: npx eslint
        language: system
        types: [javascript]

      - id: prettier
        name: Prettier
        entry: npx prettier --write
        language: system
        types: [javascript, json, yaml, markdown]

      - id: npm-test
        name: npm test
        entry: npm test
        language: system
        pass_filenames: false
        always_run: true
```

#### For Python Projects

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.2
    hooks:
      - id: gitleaks

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: check-added-large-files

  - repo: https://github.com/PyCQA/bandit
    rev: 1.7.7
    hooks:
      - id: bandit
        args: ["-c", "bandit.yaml", "-r", "."]

  - repo: https://github.com/psf/black
    rev: 24.0.0
    hooks:
      - id: black

  - repo: https://github.com/PyCQA/isort
    rev: 5.13.0
    hooks:
      - id: isort
```

#### For Terraform / IaC Projects

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.2
    hooks:
      - id: gitleaks

  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.88.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint

  - repo: https://github.com/bridgecrewio/checkov
    rev: 3.2.0
    hooks:
      - id: checkov
        files: \.(tf|tfvars)$
```

### Why Pre-Commit Hooks Are The Ultimate Shift Left

| Comparison | Pre-Commit Hook | CI/CD Scan | Production Audit |
|---|---|---|---|
| **When it finds the problem** | Before commit | After push | After deploy |
| **Time to fix** | Seconds | Minutes to hours | Days |
| **Cost** | $0 | CI minutes + human review | Incident response + reputation + compliance |
| **Blocks the developer?** | ✅ Instantly | ✅ (if enforced) | ❌ Too late |
| **Developer learns?** | Immediately contextual | Delayed, may be in different task | Post-mortem, blame-oriented |
| **Secrets in history?** | ❌ Never reaches history | ⚠️ May already be in history | ✅ Already exploited |

> **"A pre-commit hook is a security control that costs zero infrastructure, runs in under a second, and prevents the most common developer mistake: committing a secret."**

### Bypassing Pre-Commit Hooks (Anti-Pattern)

```bash
# ❌ NEVER do this
git commit -m "skip hooks" --no-verify
# This flag (--no-verify) skips ALL pre-commit hooks

# ❌ NEVER set this environment variable
SKIP=gitleaks git commit -m "skip gitleaks"
# Skips only named hooks but still dangerous
```

**Policy recommendation:** Track `--no-verify` usage:
```bash
# In your CI pipeline, check for --no-verify
if git log --grep="no-verify" --oneline | grep -q .; then
  echo "⚠️ WARNING: Someone used --no-verify"
  # Alert security team, add to audit queue
fi
```

---

## .gitignore Best Practices

> **".gitignore is your first line of defense against accidentally committing sensitive data, build artifacts, and environment-specific files."**

### What `.gitignore` Does

`.gitignore` tells Git: **"Pretend these files don't exist."**

```bash
# BEFORE .gitignore
git status
# new file:   .env              ❌ Secret!
# new file:   node_modules/     ❌ 50,000 files!
# new file:   dist/             ❌ Build output!
# new file:   .DS_Store         ❌ macOS junk!
# new file:   src/app.js        ✅ Actual code

# AFTER .gitignore (with correct patterns)
git status
# new file:   .gitignore        ✅
# new file:   src/app.js        ✅
# (secrets + junk silently ignored)
```

### Critical DevSecOps Patterns

#### Pattern 1: Ignore Secrets

```bash
# .gitignore

# ─── Environment Variables ─────────────────
.env
.env.local
.env.*.local
.env.development
.env.test
.env.production

# ─── Credentials & Keys ────────────────────
*.pem
*.key
*.p12
*.pfx
*.crt
*.der
id_rsa
id_rsa.pub
*.credentials

# ─── Config Files with Secrets ─────────────
config/credentials.yml
config/master.key          # Rails encrypted master
google-services.json       # Firebase
firebase-config.json       # Firebase
terraform.tfvars           # May contain passwords
ansible-vault-password
```

#### Pattern 2: Allow Example Files (Safe Templates)

```bash
# .gitignore
.env
.env.local
.env.*.local

# ─── BUT ───
!.env.example              # Keep safe example file
!.env.development.example  # Template for devs to copy
```

Developers copy the example:
```bash
cp .env.example .env.local
# Edit .env.local with real values (ignored by Git)
```

#### Pattern 3: Per-Directory `.gitignore`

```bash
my-project/
├── .gitignore              # Root rules
├── frontend/
│   └── .gitignore          # React-specific
│       build/
│       *.css.map
├── backend/
│   └── .gitignore          # Python-specific
│       __pycache__/
│       *.sqlite3
└── infrastructure/
    └── .gitignore          # Terraform-specific
        *.tfstate
        *.tfstate.backup
        .terraform/
```

#### Pattern 4: Keep Empty Directories

```bash
# You WANT the logs/ directory tracked, but not files inside it
mkdir logs/
echo "/*.log" > logs/.gitignore   # Ignore all .log files in this dir
touch logs/.gitkeep               # Empty file to persist directory

# Root .gitignore
logs/*
!logs/.gitkeep                    # Don't ignore .gitkeep
!logs/.gitignore                  # Don't ignore the .gitignore
```

#### Pattern 5: Normalize Line Endings

```bash
# .gitattributes (works with .gitignore for clean diffs)
* text=auto
eol=lf
*.sh text eol=lf
*.bat text eol=crlf
*.ps1 text eol=crlf
```

### `.gitignore` Template by Language

#### JavaScript / Node.js

```bash
# Dependencies
node_modules/
package-lock.json  # Optional: some teams ignore it
yarn.lock          # Optional: some teams ignore it
pnpm-lock.yaml     # Optional

# Build output
dist/
build/
.next/
out/
coverage/

# Environment
.env
.env.local
.env.production

# Logs
npm-debug.log*
yarn-debug.log*
yarn-error.log*
*.log

# IDE
.vscode/
.idea/
*.swp
.DS_Store

# Temporary
*.tmp
.cache/
```

#### Python

```bash
# Virtual environments
venv/
env/
ENV/
.venv/

# Bytecode
__pycache__/
*.py[cod]
*$py.class
*.so

# Distribution / packaging
build/
dist/
*.egg-info/

# Testing
.pytest_cache/
.tox/
.coverage
htmlcov/

# Environment
.env
.env.local

# IDE
.vscode/
.idea/
*.swp

# Jupyter
.ipynb_checkpoints/
```

#### Go

```bash
# Binary
*.exe
*.exe~
*.dll
*.so
*.dylib
main

# Test binary
*.test

# Coverage
*.out
coverage.html

# Go workspace
go.work

# Dependencies (if using vendoring)
vendor/

# IDE
.vscode/
.idea/
*.swp
```

#### Terraform

```bash
# State files (MAY CONTAIN SECRETS!)
*.tfstate
*.tfstate.*.backup
*.tfstate.backup
.terraform/
.terraform.lock.hcl

# Variables file (often has secrets!)
*.tfvars
*.tfvars.json

# Plan output
*.tfplan

# Crash logs
crash.log
crash.*.log
```

### `.gitignore` Is NOT Security

**Critical understanding:** `.gitignore` prevents *accidental* adds. It does NOT stop:

```bash
# ❌ Force-add bypasses .gitignore entirely!
git add -f .env
git commit -m "quick fix"

# ❌ .env not in .gitignore yet? It's committed!
git add .                         # .env not in .gitignore
git commit -m "add everything"    # Secret is now in history!

# ❌ .gitignore doesn't retroactively hide already-tracked files!
echo ".env" >> .gitignore
git add .gitignore
git commit -m "add gitignore"
# .env is STILL tracked because Git already knew about it!
```

**Fix for accidentally-tracked files:**
```bash
# Remove from Git tracking but keep local file
git rm --cached .env              # Unstage from Git
git add .gitignore                # Ensure .gitignore is present
git commit -m "fix: remove secret from tracking"
# Now .env is ignored and not in future commits
```

### Defense in Depth: Pre-Commit + .gitignore

| Layer | What It Catches | What It Misses |
|---|---|---|
| `.gitignore` | Accidental `git add .` | `git add -f` or missing pattern |
| Pre-commit hook (gitleaks) | Secrets even if force-added | Zero-day secret pattern |
| CI scan (gitleaks-action) | Already-committed secrets | Post-merge find |
| Runtime secret manager | Actual secret usage | Nothing — it's the last line |

Together:

```
Developer tries: git add -f .env          ← Bypasses .gitignore!
    │
    ▼
git commit -m "fix"
    │
    ▼
┌──────────────────────┐
│  pre-commit hook     │                       ← CATCHES the secret!
│  └── gitleaks        │                          (checks staged files)
│      └── EXIT 1      │
└──────────────────────┘                       ← Commit BLOCKED!
```

> **Use `.gitignore` as convenience. Use pre-commit hooks as enforcement. Use both together.**

### Checklist: Secure Git Repository Setup

```bash
# 1. Create .gitignore (secrets first)
cat > .gitignore << 'EOF'
# Secrets
.env
.env.*
*.pem
*.key
*.p12
*.pfx
*.credentials

# Dependencies
node_modules/
vendor/
__pycache__/

# Build
build/
dist/
*.exe

# Logs
*.log

# IDE
.DS_Store
.idea/
.vscode/
EOF

# 2. Create .gitattributes
cat > .gitattributes << 'EOF'
* text=auto eol=lf
*.sh text eol=lf
*.bat text eol=crlf
EOF

# 3. Install pre-commit hooks
cat > .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.2
    hooks:
      - id: gitleaks
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: check-added-large-files
      - id: detect-private-key
EOF

pip install pre-commit
pre-commit install

# 4. Commit security controls
git add .gitignore .gitattributes .pre-commit-config.yaml
git commit -m "chore: add git security controls"
```

All 20 todos are completed.

<done>


### Critical Branch Protection Rules

#### GitHub Example

Navigate to **Settings → Branches → Add rule**

```
Branch name pattern: main

☑ Require a pull request before merging
    ☐ Require approvals: 2
    ☑ Dismiss stale PR approvals when new commits are pushed
    ☑ Require review from CODEOWNERS

☑ Require status checks to pass before merging
    Status checks: [✓] ci/security-scan
                    [✓] ci/unit-tests
                    [✓] ci/build

☑ Require conversation resolution before merging

☑ Require signed commits

☑ Require linear history          (Prevents messy merges)

☑ Include administrators          (Rules apply to everyone)

☑ Restrict pushes that create files larger than 100 MB

☑ Lock branch                     (Prevents direct pushes)
```

#### GitLab Example

Navigate to **Settings → Repository → Protected branches**

```
Protect a branch: main
Allowed to merge: Maintainters + Developers (2 approvals)
Allowed to push: No one     (All changes via MR)

☑ Require approval from CODEOWNERS
☑ Require all threads to be resolved
☑ Require DCO sign-off for commits
☑ Reject non-DCO commits
☑ Require security scanning (SAST, SCA)
☑ Require pipeline to succeed
```

### Branch Protection Strategy by Branch

| Branch | Protection Level | Direct Push | Required Reviews | Status Checks |
|---|---|---|---|---|
| `main` / `master` | 🔴 Maximum | ❌ No one | ✅ 2+ | ✅ All must pass |
| `release/*` | 🔴 Maximum | ❌ No one | ✅ 2+ | ✅ All must pass |
| `develop` | 🟡 High | ❌ No one | ✅ 1 | ✅ Core checks |
| `feature/*` | 🟢 Standard | ✅ Author | ✅ 1 on PR to develop | ✅ Security scan |
| `hotfix/*` | 🟡 High | ❌ No one | ✅ 2 + emergency lead | ✅ All must pass |

---

## CI/CD Pipeline Security (Automation)

When code reaches the CI/CD pipeline, automated security tools validate it before deployment.

### Git-Based CI/CD Security Pipeline

```yaml
# .github/workflows/devsecops-git.yml
name: Git DevSecOps Pipeline

on:
  push:
    branches: [main, release/*, hotfix/*]
  pull_request:
    branches: [main, develop]

jobs:
  # ─── Gate 1: Secret Scanning ────────────────────────
  secrets-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for scan
      - name: Gitleaks Scan
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  # ─── Gate 2: Commit Signing Verification ────────────
  verify-commits:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Verify Signed Commits
        run: |
          git log --pretty=format:'%H %G?' origin/main..HEAD | while read hash status; do
            if [ "$status" != "G" ]; then
              echo "❌ Commit $hash is not GPG-signed"
              exit 1
            fi
          done
          echo "✅ All commits are GPG-signed"

  # ─── Gate 3: Code Quality & Security ────────────────
  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Semgrep SAST
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/owasp-top-ten

  # ─── Gate 4: Dependency Scanning (SCA) ──────────────
  sca:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Snyk SCA
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  # ─── Gate 5: IaC Scanning ───────────────────────────
  iac-scan:
    if: contains(toJson(github.event.pull_request.files), '.tf')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Checkov IaC Scan
        uses: bridgecrewio/checkov-action@master
        with:
          directory: ./terraform

  # ─── Gate 6: Container Security ─────────────────────
  container-scan:
    if: contains(toJson(github.event.pull_request.files), 'Dockerfile')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build Image
        run: docker build -t app:${{ github.sha }} .
      - name: Trivy Scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'app:${{ github.sha }}'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

  # ─── Gate 7: SBOM Generation ────────────────────────
  sbom:
    runs-on: ubuntu-latest
    needs: [sca]
    steps:
      - uses: actions/checkout@v4
      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          format: spdx-json
          output-file: sbom.spdx.json
      - name: Upload SBOM
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.spdx.json
```

### Pipeline Gates Explained

| Gate | Purpose | Block on Failure |
|---|---|---|
| **Secrets Scanning** | No credentials leak to history | ✅ Yes |
| **Commit Signing Verify** | All code is attributable | ✅ Yes |
| **SAST** | No code-level vulnerabilities | ✅ Yes |
| **SCA** | No vulnerable dependencies | ✅ Yes (for CRITICAL/HIGH) |
| **IaC Scan** | No cloud misconfigurations | ✅ Yes |
| **Container Scan** | No CVEs in base image | ✅ Yes (for CRITICAL) |
| **SBOM** | Track supply chain provenance | ❌ Generate artifact only |

---

## Commit Signing & Verification

Unsigned commits mean **anyone can impersonate anyone**.

### Setup GPG Signing

```bash
# 1. Generate GPG key
gpg --full-generate-key

# 2. Export public key
gpg --armor --export YOUR_KEY_ID

# 3. Add to GitHub/GitLab:
#    GitHub → Settings → SSH and GPG keys → New GPG key

# 4. Configure Git to sign all commits
git config --global user.signingkey YOUR_KEY_ID
git config --global commit.gpgsign true
git config --global tag.gpgsign true

# 5. Signed commit
git commit -S -m "add secure auth flow"

# Output:
# [main abc1234] add secure auth flow
# You need a passphrase to unlock the secret key for
# user: "Devon Developer <devon@company.com>"
# 2048-bit RSA key, ID ABC123DEF
```

### Visual Verification

```bash
# View commit signatures
git log --show-signature -5

# Output:
# commit abc1234def5678 (HEAD -> main, origin/main)
# gpg: Signature made Thu 20 Jun 2025 09:30:00 PM UTC
# gpg:                using RSA key 2B3C4D5E6F7G8H9I
# gpg: Good signature from "Devon Developer <devon@company.com>" [ultimate]
# Author: Devon Developer <devon@company.com>
# Date:   Thu Jun 20 21:30:00 2025 +0000
#
#     add secure auth flow
```

### Enforce Signed Commits

```bash
# Server-side enforcement (GitHub branch protection)
# Check UI: Settings → Branches → main → Require signed commits

# Client-side policy via pre-commit hook
# .git/hooks/pre-commit
GIT_DIR=$(git rev-parse --git-dir)
COMMIT_MSG=$(git log -1 --format=%B)

if ! git verify-commit HEAD 2>/dev/null; then
    echo "❌ Commit is not GPG-signed. Sign with: git commit -S -m '...'"
    exit 1
fi
```

---

## GitOps Security

GitOps uses Git as the **single source of truth** for infrastructure and deployments.

### GitOps Architecture

```
┌──────────────┐     git push      ┌──────────────┐     reconcile     ┌──────────┐
│  Git Repo    │ ────────────────► │  GitOps      │ ────────────────► │ Cluster  │
│  (Desired    │                   │  Agent       │                   │ (Actual) │
│   State)     │ ◄─────────────────│  (ArgoCD,    │ ◄─────────────────│          │
│              │    sync status    │   Flux)      │    observed state │          │
└──────────────┘                   └──────────────┘                   └──────────┘
         │
         │ webhook trigger
         ▼
   ┌──────────────┐
   │  CI Pipeline │ (build, test, sign)
   └──────────────┘
```

### GitOps Security Principles

| Principle | Implementation | Why It Matters |
|---|---|---|
| **Immutable source** | Git holds all config | Rollback to any previous state |
| **Auditable changes** | Every change is a commit | Full audit trail of who changed what |
| **No manual cluster edits** | All changes via Git PR | Prevents configuration drift |
| **Rollback** | `git revert` restores last known good state | Fast recovery from bad deployments |
| **Drift detection** | GitOps agent continuously reconciles | Unauthorized manual changes detected |

### GitOps Security Threats

| Threat | Attack Vector | Mitigation |
|---|---|---|
| **Compromised GitOps agent** | Attacker gains access to ArgoCD pod | Run agent in dedicated namespace, RBAC, network policies |
| **Malicious manifest commit** | Backdoor in Kubernetes deployment YAML | SAST_manifest scanning in CI, signed commits, PR required |
| **Git repo compromise** | Attacker pushes directly to config repo | Branch protection, signed commits, 2FA required |
| **Webhook interception** | MITM on Git → GitOps webhook | TLS 1.3, mutual TLS, webhook secret verification |
| **Secret leakage in GitOps** | Kubernetes secrets stored in Git | Use sealed-secrets or external secret managers |

### Sealed Secrets (Encrypting Secrets for GitOps)

```bash
# Install kubeseal
brew install kubeseal

# Create a normal Kubernetes secret
cat <<EOF > mysecret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-password
type: Opaque
stringData:
  password: SuperSecret123!
EOF

# Encrypt with the cluster's public key
kubeseal --controller-namespace=kube-system \
         --controller-name=sealed-secrets \
         --format yaml < mysecret.yaml > sealed-secret.yaml

# Result: safe to commit to Git!
cat sealed-secret.yaml
# apiVersion: bitnami.com/v1alpha1
# kind: SealedSecret
# metadata:
#   name: db-password
# spec:
#   encryptedData:
#     password: AgByA...  (encrypted data only cluster can decrypt)
```

---

## Secrets Management in Git

### The Problem

Git history is **immutable and forever**.

```bash
# Developer makes a mistake
git add config/.env
git commit -m "add config"
git push origin main

# Even if you "fix" it:
git rm config/.env
git commit -m "remove secret"
git push

# 🔴 The secret is still in Git history!
git log --all --full-history -- config/.env
# commit abc123  -- still contains the secret!
```

### Prevention: Pre-Commit Rules

```bash
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks:
      - id: detect-secrets
        args: ["--baseline", ".secrets.baseline"]

  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.2
    hooks:
      - id: gitleaks
        entry: gitleaks protect --verbose --redact --staged
```

### Recovery: If a Secret Is Already in Git

```bash
# Step 1: Immediately rotate the secret!
aws iam update-access-key --access-key-id AKIAIOSFODNN7EXAMPLE \
    --status Inactive --user-name dev-user

# Step 2: Remove from history (CAUTION: rewrites history!)
git filter-repo --path config/.env --invert-paths
# OR: BFG Repo-Cleaner
bfg --delete-files config/.env

# Step 3: Force push to overwrite history
git push --force-with-lease origin main

# Step 4: Notify all fork owners
# Step 5: Audit cloud/access logs for abuse
```

**⚠️ Warning:** Rewriting history on shared branches is dangerous. Coordinate with the team.

### Safe Alternatives to Committing Secrets

| Approach | Tool | When to Use |
|---|---|---|
| Environment variables (local) | `.env` + `.gitignore` | Development only |
| Secret manager (cloud-native) | AWS Secrets Manager, Azure Key Vault | Production workloads |
| Git-backed secret encryption | Mozilla SOPS, git-crypt | Teams needing secrets in Git |
| Sealed secrets (GitOps) | Bitnami Sealed Secrets | Kubernetes + GitOps workflows |
| Vault integration | HashiCorp Vault | Enterprise, dynamic secrets |

---

## Supply Chain Security via Git

Git is the foundation of software supply chain security.

### Signed Commits = Attestation

```bash
# Tag a verified release
git tag -s v1.2.3 -m "Release v1.2.3"
git push origin v1.2.3

# Verify before deploying
git verify-tag v1.2.3
# gpg: Signature made Thu 20 Jun 2025
# gpg: Good signature from "Release Manager <releases@company.com>"
```

### SBOMs (Software Bill of Materials)

```bash
# Generate SBOM from your repo
syft packages dir:. -o spdx-json > sbom.spdx.json

# Sign the SBOM
gpg --detach-sign sbom.spdx.json

# Store both in Git release assets
gh release create v1.2.3 \
  sbom.spdx.json \
  sbom.spdx.json.sig \
  --title "v1.2.3" \
  --notes "Signed release with SBOM"
```

### SLSA (Supply Chain Levels for Software Artifacts)

| Level | Requirement | Git Role |
|---|---|---|
| **1** | Build process fully scripted | CI/CD pipeline defined in `.github/workflows/` |
| **2** | Build runs on hardened builder | GitHub Actions / GitLab CI runners |
| **3** | Build is hermetic, reproducible | Pin dependencies by hash, lock files committed |
| **4** | Two-party review + hermetic | Signed commits + branch protection + required reviewers |

---

## Git Security Tooling

| Tool | Purpose | When to Use |
|---|---|---|
| **Git Hooks** | Local validation | Block before commit |
| **Gitleaks** | Secret detection | Pre-commit + CI |
| **TruffleHog** | Secret detection (deep scan) | CI with full history |
| **GitGuardian** | Secret monitoring | SaaS, real-time repo monitoring |
| **Sigstore / Cosign** | Signed artifacts | Container image signing |
| **Trivy** | Container + Git scanning | CI pipeline |
| **Sealed Secrets** | Encrypt secrets for Git | GitOps Kubernetes |
| **Mozilla SOPS** | Encrypt files in Git | General encrypted files in Git |
| **Git-crypt** | Transparent encryption | Team-shared encrypted files |
| **GitHub Advanced Security** | Built-in secret + code scanning | GitHub Enterprise users |
| **GitLab Secure** | SAST, DAST, SCA, secrets | GitLab Ultimate users |

---

## Quick Reference: Secure Git Workflow

### Daily Developer Workflow

```bash
# 1. Start with latest main
git checkout main
git pull origin main

# 2. Create feature branch
git checkout -b feature/add-auth

# 3. Make changes, stage
git add src/auth.js

# 4. Pre-commit hooks run automatically
#    (secrets scan, lint, SAST, etc.)
git commit -S -m "feat: add OAuth2 authentication"

# 5. Push branch to origin (triggers CI)
git push -u origin feature/add-auth

# 6. Open Pull Request
#    - Requires 2 reviews
#    - CI checks must pass
#    - Signed commits verified
#    - CODEOWNERS approval required

# 7. After merge, delete branch
git branch -d feature/add-auth
```

### Repository Setup Checklist

```markdown
## Repo Security Setup

- [ ] Pre-commit hooks installed (gitleaks, bandit, checkov)
- [ ] Branch protection on `main` (2 reviews, status checks, signed commits)
- [ ] CODEOWNERS file for critical paths
- [ ] `.gitignore` for secrets, `.env`, build artifacts
- [ ] CI pipeline: SAST + SCA + IaC scan + container scan
- [ ] Commit signing required for all contributors
- [ ] Dependency update automation (Dependabot/ Renovate)
- [ ] `.secrets.baseline` for detect-secrets
- [ ] Emergency response plan for leaked secrets
```

---

## Summary

| Security Gate | Tool/Control | Goal |
|---|---|---|
| **Local** | Pre-commit hooks (gitleaks, bandit) | Block secrets before they enter history |
| **Remote** | Branch protection + required reviews | Never ship unreviewed code |
| **CI/CD** | SAST + SCA + container scan | Automated validation of every change |
| **Deployment** | Signed commits + signed artifacts | Attributable, auditable releases |
| **Operations** | GitOps + drift detection | Infrastructure state matches Git state |

> **In DevSecOps, Git is not just version control — it is the security backbone of your entire software delivery pipeline.**

Every commit is an attack surface. Every PR is a security boundary. Every merge is a trust decision. Treat Git accordingly.

---

*Generated for DevSecOps teams securing Git workflows and CI/CD pipelines.*
