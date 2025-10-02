# Git Best Practices & SOP (Simplified GitFlow + Jira-first commits)

> **Scope:** Small Scrum team | Jira for work tracking | Azure DevOps (Repos/PRs/policies)
> **Key decision:** Commits **must** include a Jira issue key in the **title** (subject line). Smart-Commit commands go in the **body/footers**.

---

## Branching Strategy (Simplified GitFlow)

**Long-lived**

* `main` – production-ready. Hotfixes and releases land here.
* `staging` – integration for the next release (PR target for normal work).

**Short-lived**

* `feature/ABC-123-short-slug` – new feature (branch from `staging`).
* `fix/ABC-456-bug-slug` – non-urgent fix (branch from `staging`).
* `hotfix/ABC-789-short-slug` – urgent prod fix (branch from `main`).

**Flow**

1. Feature/fix: branch from `staging` → PR to `staging` → CI → **Squash merge**.
2. Release: PR `staging` → `main` → tag `vMAJOR.MINOR.PATCH`.
3. Hotfix: branch from `main` → PR to `main` → tag (e.g., `v1.3.1`) → **back-merge** into `staging`.

*Rationale:* Gives you **staging** isolation for planned releases while keeping a near-trunk cadence and fast hotfixes. ([learn.microsoft.com][1])

---

## Commit Messages

### 1) Requirements

* **Jira key in the title (subject line)** for every commit. This ensures clean traceability in Jira and makes the key visible in changelogs. Atlassian links commits to issues whenever the **issue key** appears in the message; putting it in the title guarantees visibility. ([Atlassian Support][2])
* Use **Conventional Commits** for the header so we can auto-generate release notes.
  Format (**with key at end of title** so parsers remain happy):

  ```
  <type>(optional-scope): <short summary> (ABC-123)
  ```

  > Why “key at end”? Conventional parsers expect the header to start with `<type>(scope): ...`. Adding the Jira key **at the end of the subject** keeps both: (a) machine-parsable Conventional Commits and (b) Jira traceability. Atlassian’s linking works anywhere in the message. ([Atlassian Support][2])

### 2) What is `optional-scope`? (with examples)

`scope` is a short label for the area touched. Use stable, team-agreed labels (e.g., `auth`, `api/user`, `ui/forms`, `build/ci`). Examples:

* `feat(auth): add TOTP enrollment (ABC-123)`
* `fix(api/user): return 404 for unknown id (ABC-456)`
* `refactor(ui/forms): extract PasswordField (ABC-471)`
* `test(auth/login,auth/register): edge cases (ABC-482)`
* `chore(build/ci): cache node_modules (ABC-490)`

### 3) Description/body rules (for changelogs)

* **Subject** (the title) ≤ 72 chars, imperative, no trailing period.
* **Body**: explain **what** and **why** (not every “how”); wrap at ~72 chars.
* **Footers**:

  * `BREAKING CHANGE: <impact + migration>`
  * `Refs: ABC-123` (redundant but helpful for search)

*(Conventional Commit parsing + Jira linking + readable changelogs.)* ([DEV Community][3])

### 4) Smart Commits (default: enabled in our templates)

Place **Smart-Commit commands in the body/footers** (not in the header) so Conventional Commits parsing is unaffected. Examples:

```
ABC-123 #comment Implemented verify endpoint
ABC-123 #time 1h
ABC-123 #transition In Review
```

> Requires Jira–Azure DevOps integration (e.g., **Azure DevOps for Jira** app) and Smart Commits enabled in Jira. ([Atlassian Support][4])

### 5) When to commit

* **Commit early and often**: each **logical step** that builds/tests locally (≈30–120 min of work).
* Push regularly; keep feature branches **short-lived** and PRs small.
  *(Improves review quality and reduces merge conflicts.)*

---

## Commit Message Templates

### A) **Full template** (copy into `docs/commit-template.txt` and set `git config commit.template docs/commit-template.txt`)

```txt
# Title (subject)
# Keep ≤72 chars, imperative; Conventional Commit + Jira key in title:
# <type>(scope): <summary> (ABC-123)
feat(auth): add TOTP login (ABC-123)

# Body (what + why, wrapped at ~72 cols)
Adds a 6-digit TOTP factor to augment password login.
Improves security and aligns with RFC 6238-compatible apps.

- New /auth/totp endpoints: enroll, verify
- UI: setup flow behind feature flag
- Docs updated

# Footers (Smart Commits + references)
ABC-123 #comment Implemented enrollment + verify endpoints
ABC-123 #time 1h
ABC-123 #transition In Review
Refs: ABC-123
# BREAKING CHANGE: users must enroll a TOTP device on next login
```

### B) **Bug fix example**

```txt
fix(api/user): return 404 for unknown id (ABC-456)

Return 404 with code USER_NOT_FOUND instead of 500.

ABC-456 #comment Adjusted controller + contract test
ABC-456 #time 30m
Refs: ABC-456
```

---

## Pull Requests

### Targets

* Default PRs → `staging`
* Hotfix PRs → `main` (then back-merge to `staging`)

### Merge strategy

* **Squash merge** only. On completion, **edit the squash commit message** so the final commit on `staging`/`main` follows our header rule and keeps the Jira key in the **title**. Azure DevOps lets you change this at “Complete” time. ([learn.microsoft.com][5])

**Recommended squash message for PR completion**

```
feat(auth): add TOTP login (ABC-123)

See PR #<id>. Includes UI setup flow (flagged) and API endpoints.

ABC-123 #comment Merged to staging
Refs: ABC-123
```

### PR description template (with Smart Commits)

Create `.azuredevops/pull_request_template.md`:

```md
## Summary
What changed and why?
Linked Jira: ABC-123

## Test Notes
- [ ] Unit tests updated/added
- [ ] Manual verification steps (screenshots if UI)

## Risk & Rollback
Impact, known risks, and quickest rollback.

## Release Notes (CHANGELOG one-liner)
feat(auth): add TOTP login (ABC-123)

---
## Smart Commits
ABC-123 #comment PR opened for review
```

Azure DevOps supports PR templates and editing the completion message; use both to keep subjects consistent. ([learn.microsoft.com][5])

---

## Pre-Commit Checklist (Developer)

* [ ] Builds locally; tests pass.
* [ ] Lint/format clean.
* [ ] No secrets or large binaries committed.
* [ ] **Title** follows: `<type>(scope): <summary> (ABC-123)`.
* [ ] Body explains **what/why**.
* [ ] **Smart-Commit** lines in body/footers as needed (comment, time, transition).
* [ ] Branch name contains Jira key (e.g., `feature/ABC-123-short-slug`).

*(Automate with `pre-commit`, `commitlint`, and a secrets scanner.)*

---

# Azure DevOps Branch Policy Checklist

Apply the following to both `main` and `staging`:
-
- [ ] **Require pull request to merge** (no direct pushes).
- [ ] **Minimum number of reviewers**: 1 (increase later if needed).
- [ ] **Build validation**: CI pipeline must succeed before completion.
- [ ] **Merge strategy**: Only allow **Squash merge**.
- [ ] **Automatically include reviewers**: add defaults per area if desired.
- [ ] **Automatically delete source branch** after merge.
- [ ] **Limit who can bypass policies**: only admins.
- [ ] **Status checks**: enable lint/test/security scan if available.
- [ ] **Check for linked work items**: PR must reference Jira issue key in title.

Recommended extras:
- [ ] Enable **comment requirements** if AI bot reviewer is integrated.
- [ ] Add **required reviewers** for sensitive areas (e.g., security-critical code).

---

## Repository & Policy Setup (once)

* Protect `main` and `staging` with branch policies:

  * Require PR; min 1 reviewer; build validation; allow **Squash**; delete branch on completion. ([learn.microsoft.com][1])
* Add `.azuredevops/pull_request_template.md`.
* Configure `git config commit.template docs/commit-template.txt`.
* Connect Jira ↔ Azure DevOps (install **Azure DevOps for Jira**), enable **Smart Commits**. ([Atlassian Support][4])

---

## Notes & Rationale

* **Why put the Jira key in the title?** Atlassian links on any occurrence of the key; putting it in the subject ensures it shows prominently in changelogs and in `git log --oneline`. ([Atlassian Support][2])
* **Why keep Smart-Commit commands out of the title?** To avoid breaking Conventional Commit parsers and to keep titles short; Atlassian parses commands from the body. ([Atlassian Support][6])
* **Why squash?** Cleaner history on `staging`/`main` while preserving full discussion in the PR. You can (and should) **edit the final message** at merge time. ([learn.microsoft.com][5])

---
## Resources

* **Pro Git Book (PDF)**: https://github.com/progit/progit2/releases/latest/progit.pdf 
* **Pro Git Book (HTML)**: https://git-scm.com/book/en/v2
* **Learn Git Branching (interactive)**: https://learngitbranching.js.org/

---
## References

[1]: https://learn.microsoft.com/en-us/azure/devops/repos/git/pull-requests?view=azure-devops&utm_source=chatgpt.com "Create a pull request to review and merge code"
[2]: https://support.atlassian.com/jira-software-cloud/docs/reference-issues-in-your-development-work/?utm_source=chatgpt.com "Reference work items in your development projects | Jira ..."
[3]: https://dev.to/itxshakil/commit-like-a-pro-a-beginners-guide-to-conventional-commits-34c3?utm_source=chatgpt.com "Commit Like a Pro: A Beginner's Guide to Conventional ..."
[4]: https://support.atlassian.com/jira-cloud-administration/docs/integrate-azure-devops-with-jira/?utm_source=chatgpt.com "Connect Azure DevOps to Jira"
[5]: https://learn.microsoft.com/en-us/azure/devops/repos/git/pull-request-templates?view=azure-devops&utm_source=chatgpt.com "Improve pull request descriptions using templates"
[6]: https://support.atlassian.com/jira-software-cloud/docs/process-issues-with-smart-commits/?utm_source=chatgpt.com "Process work items with smart commits | Jira Cloud"
