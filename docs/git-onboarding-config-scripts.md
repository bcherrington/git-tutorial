# Git Onboarding Config Snippets (Copy‑Paste Pack)

A single place to copy the exact commands and files you need. Adapt placeholders like `ABC-123`, `{org}`, `{project}`, `{repoId}` as required.

---

## 1) Developer Git Setup (one‑time)

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
  echo "\nERROR: Commit title must match:\n  <type>(scope): <summary> (ABC-123)\nExamples:\n  feat(auth): add TOTP login (ABC-123)\n  fix(api/user): return 404 for unknown id (ABC-456)\n" >&2
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
    "@commitlint/cli": "^19.0.0",
    "@commitlint/config-conventional": "^19.0.0",
    "husky": "^9.0.0"
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
  extends: ['@commitlint/config-conventional'],
  rules: {
    // Require Jira key at end of subject line: (ABC-123)
    'subject-case': [2, 'always', ['sentence-case', 'lower-case', 'start-case', 'pascal-case']],
    'subject-full-stop': [2, 'never', '.'],
    'subject-empty': [2, 'never'],
    // Custom regex: allow anything, but must end with (AAA-123)
    'subject-match': [2, 'always', /.*\([A-Z]{2,}-\d+\)$/],
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
    rev: v4.6.0
    hooks:
      - id: check-yaml
      - id: end-of-file-fixer
      - id: trailing-whitespace

  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
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

## 5) Azure Pipelines (Build Validation CI)

A single pipeline that tries common ecosystems (Node, Python, .NET). It’s safe to keep all; steps run only if the files exist.

**azure-pipelines.yml**

```yaml
trigger:
  branches:
    include: [ main, staging, feature/*, fix/*, hotfix/* ]

pr:
  branches:
    include: [ main, staging ]

pool:
  vmImage: ubuntu-latest

steps:
  # Node.js projects
  - script: |
      if [ -f package.json ]; then
        echo "Detected Node project"
        corepack enable || true
        npm ci
        npm run -s lint || echo "No lint script"
        npm test --silent || echo "No tests"
      else
        echo "No package.json; skipping Node"
      fi
    displayName: "Node: install, lint, test"

  # Python projects
  - task: UsePythonVersion@0
    inputs: { versionSpec: '3.x', addToPath: true }
    displayName: "Python: set version"
  - script: |
      if [ -f requirements.txt ]; then
        echo "Detected Python project"
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        if python -c "import pytest" 2>/dev/null; then pytest -q || true; else echo "pytest not installed"; fi
      else
        echo "No requirements.txt; skipping Python"
      fi
    displayName: "Python: deps & tests"

  # .NET projects
  - script: |
      if ls -1 *.sln 2>/dev/null | grep -q "."; then
        echo "Detected .NET solution"
        dotnet restore
        dotnet build --configuration Release --nologo
        dotnet test  --configuration Release --no-build --nologo
      else
        echo "No .sln; skipping .NET"
      fi
    displayName: ".NET: restore, build, test"
```

> Place this file at the repo root. Use it as the **Build validation** pipeline for `main` and `staging` branch policies.

---

## 6) Add Build Validation to Branch Policies (UI steps)

1. **Project Settings › Repos › Repositories** → select your repo → **Policies**.
2. Under **Branch policies**, select `main` → **+ Add build policy** → pick the above pipeline (e.g., *CI: Lint & Test*).
3. Repeat for `staging`.
4. Also enable: *Require a minimum number of reviewers = 1*, *Check for linked work items*, *Limit merge types to Squash*, *Delete source branch on completion*.

> Optional: Require specific reviewers for sensitive paths.

---

## 7) PR Template (reference)

Create **.azuredevops/pull_request_template.md** (already provided separately). Azure DevOps auto‑loads this into new PRs.

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
## Smart Commits
ABC-123 #comment PR opened for review
```

---

## 8) AI Pre‑Review: Post Comments to PR (template)

If you use an AI tool that outputs findings to stdout, you can post them as a comment via Azure DevOps REST using the system access token in PR validation builds.

> **Note:** This example assumes your pipeline has variables for `{org}`, `{project}`, `{repoId}`, and the PR ID as `$(System.PullRequest.PullRequestId)` when running in PR context.

**Azure Pipelines step (bash) to post a single summary comment):**

```yaml
- script: |
    set -euo pipefail
    PR_ID="$(System.PullRequest.PullRequestId)"
    ORG="{org}"       # e.g., myorg
    PROJECT="{project}"  # e.g., MyProject
    REPO_ID="{repoId}"   # GUID or name

    # Your AI tool here; write markdown to review.md
    echo "### AI Review Summary\n\n- No critical issues found." > review.md

    API_URL="https://dev.azure.com/${ORG}/${PROJECT}/_apis/git/repositories/${REPO_ID}/pullRequests/${PR_ID}/threads?api-version=7.0"

    BODY=$(jq -n --arg txt "$(sed 's/"/\\"/g' review.md)" '{comments:[{parentCommentId:0,content:$txt,commentType:1}],status:1}')

    curl -sS -X POST "$API_URL" \
      -H "Content-Type: application/json" \
      -H "Authorization: Bearer $(System.AccessToken)" \
      -d "$BODY"
  env:
    SYSTEM_ACCESSTOKEN: $(System.AccessToken)
  displayName: "AI Review → Post PR comment"
```

> Ensure **Allow scripts to access the OAuth token** is enabled in the pipeline. Replace the AI step with your tool; keep the POST as shown.

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

1. **PR template** + **branch policies** + **azure-pipelines.yml**
2. **Commit template** + minimal **commit-msg hook** (section 2)
3. Secrets scanning via **pre-commit** (section 4)
4. Optional **commitlint + Husky** (section 3)
5. AI pre‑review posting to PR (section 8)

---

**Done.** Paste these files/blocks into your repo and policies; the team can start today with a predictable, traceable flow.
