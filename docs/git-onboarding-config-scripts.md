# Git Onboarding Config Snippets (Copy‑Paste Pack)

A single place to copy the exact commands and files you need for **GitHub**. Adapt placeholders like `ABC-123`, `<org>`, `<repo>` as required.

---

## 1) Developer Git Setup (one‑time)

### 1a) SSH Key Setup for GitHub

```bash
# Generate SSH key (if you don't have one)
ssh-keygen -t ed25519 -C "your.email@company.com"

# Start ssh-agent and add your key
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# Copy public key to clipboard
# macOS:
pbcopy < ~/.ssh/id_ed25519.pub
# Linux (with xclip):
# xclip -selection clipboard < ~/.ssh/id_ed25519.pub
# Windows (Git Bash):
# clip < ~/.ssh/id_ed25519.pub
# Or display it to copy manually:
cat ~/.ssh/id_ed25519.pub

# Add to GitHub: Settings → SSH and GPG keys → New SSH key
# Paste the public key and save

# Test connection
ssh -T git@github.com
```

### 1b) Git Configuration

```bash
# Identify yourself
git config --global user.name  "Your Name"
git config --global user.email "your.email@company.com"

# Cleaner history: rebase on pull (optional but recommended)
git config --global pull.rebase true

# Default initial branch name
git config --global init.defaultBranch main

# Use the repo's commit template globally (optional)
mkdir -p ~/.git-templates
cp docs/commit-template.txt ~/.git-templates/commit-template.txt
git config --global commit.template ~/.git-templates/commit-template.txt
```

**Per‑repo (always set in each repo):**

```bash
# Ensure this repo uses the checked-in template
git config commit.template docs/commit-template.txt
```

---

## 2) Minimal Commit Message Enforcement (no Node.js required)

Create a lightweight `commit-msg` hook that checks:

* Conventional Commits header pattern
* Jira key in the **title** (e.g., `(ABC-123)`)

> Save as `.git/hooks/commit-msg` and make it executable: `chmod +x .git/hooks/commit-msg`

```bash
#!/usr/bin/env bash
set -euo pipefail
MSG_FILE="$1"
TITLE=$(head -n1 "$MSG_FILE")

# Conventional header: <type>(optional-scope): <summary> (ABC-123)
# Types: feat|fix|docs|refactor|test|chore|ci|build|perf|revert
HEADER_RE='^(feat|fix|docs|refactor|test|chore|ci|build|perf|revert)(\([^)]+\))?: .+ \([A-Z]{2,}-[0-9]+\)$'

if ! echo "$TITLE" | grep -Eq "$HEADER_RE"; then
  echo "" >&2
  echo "ERROR: Commit title must match:" >&2
  echo "  <type>(scope): <summary> (ABC-123)" >&2
  echo "" >&2
  echo "Examples:" >&2
  echo "  feat(auth): add TOTP login (ABC-123)" >&2
  echo "  fix(api/user): return 404 for unknown id (ABC-456)" >&2
  echo "" >&2
  exit 1
fi
```

> Tip: If you prefer Node-based tooling (commitlint + Husky), see section **3)**.

---

## 3) Commitlint + Husky (Node projects)

**package.json** (add to devDependencies/scripts):

```json
{
  "devDependencies": {
    "@commitlint/cli": "^19.5.0",
    "@commitlint/config-conventional": "^19.5.0",
    "commitlint-plugin-jira-rules": "^1.8.0",
    "husky": "^9.1.0"
  },
  "scripts": {
    "prepare": "husky",
    "lint:commits": "commitlint --from=origin/main --to=HEAD"
  }
}
```

**commitlint.config.cjs** (Jira key at end of title + conventional header):

```js
module.exports = {
  plugins: ['commitlint-plugin-jira-rules'],
  extends: ['@commitlint/config-conventional'],
  rules: {
    // Require Jira key at end of subject line: (ABC-123)
    'subject-case': [2, 'always', ['sentence-case', 'lower-case', 'start-case', 'pascal-case']],
    'subject-full-stop': [2, 'never', '.'],
    'subject-empty': [2, 'never'],
    // Use jira-rules plugin to validate Jira key format
    'jira-task-id-project-key': [2, 'always', ['ABC', 'XYZ']], // Replace with your Jira project keys
    'jira-task-id-separator': [2, 'always', '-'],
    'jira-task-id-max-length': [2, 'always', 9],
  },
};
```

**Alternative: Simple regex validation without plugin**

```js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  parserPreset: {
    parserOpts: {
      headerPattern: /^(\w+)(?:\(([^)]*)\))?: (.+) \(([A-Z]{2,}-\d+)\)$/,
      headerCorrespondence: ['type', 'scope', 'subject', 'ticket'],
    },
  },
  rules: {
    'subject-case': [2, 'always', ['sentence-case', 'lower-case', 'start-case', 'pascal-case']],
    'subject-full-stop': [2, 'never', '.'],
    'subject-empty': [2, 'never'],
  },
};
```

**Enable Husky hooks**

```bash
npx husky init
```

This creates `.husky/pre-commit`. Add a **commit-msg** hook:

```bash
cat > .husky/commit-msg << 'EOF'
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npx --no-install commitlint --edit "$1"
EOF
chmod +x .husky/commit-msg
```

---

## 4) Pre‑commit Framework (polyglot)

**.pre-commit-config.yaml**

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: check-yaml
      - id: end-of-file-fixer
      - id: trailing-whitespace
      - id: check-added-large-files
      - id: check-merge-conflict

  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

**Install & activate**

```bash
pip install pre-commit detect-secrets  # or use your env manager
pre-commit install
# Initialize baseline once (creates .secrets.baseline)
detect-secrets scan > .secrets.baseline
```

> Add `.secrets.baseline` to the repo. Update it when legitimate secrets patterns change.

---

## 5) GitHub Actions (Build Validation CI)

A single workflow that tries common ecosystems (Node, Python, .NET). It’s safe to keep all; steps run only if the files exist.

**.github/workflows/ci.yml**

```yaml
name: CI

on:
  push:
    branches: [ main, staging, 'feature/**', 'fix/**', 'hotfix/**' ]
  pull_request:
    branches: [ main, staging ]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      # Node.js projects
      - name: Setup Node.js
        if: hashFiles('package.json') != ''
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Node - Install dependencies
        if: hashFiles('package.json') != ''
        run: |
          if [ -f package-lock.json ]; then
            npm ci
          else
            npm install
          fi

      - name: Node - Lint
        if: hashFiles('package.json') != ''
        run: npm run lint || echo "No lint script"
        continue-on-error: true

      - name: Node - Test
        if: hashFiles('package.json') != ''
        run: npm test

      # Python projects
      - name: Setup Python
        if: hashFiles('requirements.txt') != '' || hashFiles('pyproject.toml') != ''
        uses: actions/setup-python@v5
        with:
          python-version: '3.x'
          cache: 'pip'

      - name: Python - Install dependencies
        if: hashFiles('requirements.txt') != ''
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Python - Test
        if: hashFiles('requirements.txt') != ''
        run: |
          if python -c "import pytest" 2>/dev/null; then
            pytest -q
          else
            echo "pytest not installed"
          fi
        continue-on-error: true

      # .NET projects
      - name: Setup .NET
        if: hashFiles('**/*.sln') != ''
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.x'

      - name: .NET - Restore
        if: hashFiles('**/*.sln') != ''
        run: dotnet restore

      - name: .NET - Build
        if: hashFiles('**/*.sln') != ''
        run: dotnet build --configuration Release --no-restore

      - name: .NET - Test
        if: hashFiles('**/*.sln') != ''
        run: dotnet test --configuration Release --no-build --verbosity normal
```

> Place this file at `.github/workflows/ci.yml`. Configure branch protection rules to require this workflow to pass before merging.

---

## 6) GitHub Branch Protection Rules (UI steps)

1. Go to **Settings** → **Branches** → **Add branch protection rule**
2. **Branch name pattern**: `main`
3. Enable the following:
   - ✅ **Require a pull request before merging**
     - Require approvals: **1**
     - Dismiss stale pull request approvals when new commits are pushed
   - ✅ **Require status checks to pass before merging**
     - Require branches to be up to date before merging
     - Add status check: **test** (from the CI workflow)
   - ✅ **Require conversation resolution before merging**
   - ✅ **Do not allow bypassing the above settings**
4. **Save changes**
5. Repeat for `staging` branch

**Optional: CODEOWNERS for automatic review requests**

Create `.github/CODEOWNERS`:

```
# Default owners for everything
* @your-team

# Specific paths
/docs/ @tech-writers
*.py @python-team
/.github/ @devops-team
```

---

## 7) PR Template (reference)

Create **.github/pull_request_template.md** (already provided separately). GitHub auto‑loads this into new PRs.

```markdown
## Summary
- What changed and why?
- Link Jira: ABC-123

## Test Notes
- [ ] Unit tests updated/added
- [ ] Manual steps / screenshots (if UI)

## Risk & Rollback
- Impact / risks
- Rollback plan

## Release Notes (CHANGELOG one‑liner)
feat(auth): add TOTP login (ABC-123)

---
## Smart Commits (if using Jira integration)
ABC-123 #comment PR opened for review
```

> The template file is located at `.github/pull_request_template.md` in this repository.

---

## 8) AI Pre‑Review: Post Comments to PR (GitHub Actions)

If you use an AI tool that outputs findings to stdout, you can post them as a comment via GitHub API using the built-in `GITHUB_TOKEN`.

> **Note:** This example runs in PR context and uses GitHub Actions environment variables.

**GitHub Actions workflow step to post a PR comment:**

```yaml
- name: Run AI Review
  id: ai-review
  run: |
    # Your AI tool here; write markdown to review.md
    echo "### AI Review Summary" > review.md
    echo "" >> review.md
    echo "- No critical issues found." >> review.md
    echo "- Code quality looks good." >> review.md

- name: Post AI Review Comment
  if: github.event_name == 'pull_request'
  uses: actions/github-script@v7
  with:
    script: |
      const fs = require('fs');
      let reviewContent;
      try {
        reviewContent = fs.readFileSync('review.md', 'utf8');
      } catch (error) {
        console.log('No review file found, skipping comment');
        return;
      }

      await github.rest.issues.createComment({
        owner: context.repo.owner,
        repo: context.repo.repo,
        issue_number: context.issue.number,
        body: reviewContent
      });
```

**Alternative: Using curl directly**

```yaml
- name: Post AI Review Comment (curl)
  if: github.event_name == 'pull_request'
  run: |
    REVIEW_CONTENT=$(cat review.md | jq -Rs .)
    BODY=$(jq -n --argjson body "$REVIEW_CONTENT" '{body: $body}')

    curl -sS -X POST \
      -H "Authorization: Bearer ${{ secrets.GITHUB_TOKEN }}" \
      -H "Content-Type: application/json" \
      -d "$BODY" \
      "https://api.github.com/repos/${{ github.repository }}/issues/${{ github.event.pull_request.number }}/comments"
```

> The `GITHUB_TOKEN` is automatically available in GitHub Actions. No additional configuration needed.

---

## 9) Release Tagging (SemVer) Commands

```bash
# After PR from staging → main is completed and CI is green
git switch main && git pull --ff-only

# Tag (example: minor release)
git tag -a v1.3.0 -m "Release 1.3.0"
git push origin v1.3.0
```

---

## 10) Quick‑ref: Conventional Commit Title Examples (with Jira key)

```
feat(auth): add TOTP enrollment (ABC-123)
fix(api/user): return 404 for unknown id (ABC-456)
refactor(ui/forms): extract PasswordField (ABC-471)
chore(build/ci): cache pnpm (ABC-490)
```

---

## 11) Team Adoption Order (lowest friction first)

1. **SSH setup** (section 1a) + **Git config** (section 1b)
2. **PR template** (section 7) + **branch protection rules** (section 6) + **GitHub Actions CI** (section 5)
3. **Commit template** + minimal **commit-msg hook** (section 2)
4. Secrets scanning via **pre-commit** (section 4)
5. Optional **commitlint + Husky** (section 3)
6. AI pre‑review posting to PR (section 8)

---

**Done.** Paste these files/blocks into your repo and configure GitHub settings; the team can start today with a predictable, traceable flow.
