# 🚀 Migration Guide: Azure DevOps → GitHub

We are migrating the **OneSync** repository from **Azure DevOps** to **GitHub**.  
- **GitHub (`origin`)** → new default remote  
- **Azure DevOps (`devops`)** → temporary backup remote (to be retired)

---

## 1. Switch Remotes

Run these commands in your repo:

```bash
# Rename Azure DevOps remote → 'devops'
git remote rename origin devops

# Add GitHub as new 'origin'
git remote add origin https://github.com/modena-aec/OneSync.git

# Verify remotes
git remote -v
````

Expected:

```
origin  https://github.com/modena-aec/OneSync.git (fetch/push)
devops  https://dev.azure.com/Modena-AEC/_git/OneSync (fetch/push)
```

---

## 2. Sync GitHub with Azure (one-time)

Make sure GitHub has the **full history**:

```bash
# Fetch everything from Azure
git fetch devops --prune --tags

# Push all branches
git push origin --all

# Push all tags
git push origin --tags
```

---

## 3. During Transition

**Preferred:** Push to GitHub only.
If Azure still needs updates, push explicitly:

```bash
git push origin main     # GitHub
git push devops main     # Azure (temporary)
```

*(Optional dual-push, not recommended long-term)*

```bash
git remote set-url --add --push origin https://github.com/modena-aec/OneSync.git
git remote set-url --add --push origin https://dev.azure.com/Modena-AEC/_git/OneSync
```

---

## 4. Final Cut-over (when Azure shuts down)

```bash
# Remove Azure push URL (if dual-push was set)
git remote set-url --push --delete origin https://dev.azure.com/Modena-AEC/_git/OneSync

# Remove Azure remote entirely
git remote remove devops

# Clean up
git remote prune origin
```

---

## 5. IDE Instructions

### Visual Studio Code

* Open **Source Control (Ctrl+Shift+G)** → `…` → **Remote Repositories → Add Remote**.
* Paste the GitHub URL.
* Rename/remove the Azure remote in the integrated terminal.

👉 Docs: [VS Code Git Overview](https://code.visualstudio.com/docs/sourcecontrol/overview)

---

### Visual Studio (full IDE)

* **View → Git Repository (or Team Explorer)** → **Repository Settings → Remotes**
* Rename `origin` → `devops`
* Add new `origin` with GitHub URL
* Use **Sync** to push branches & tags.

👉 Docs: [Manage Git Repos in Visual Studio](https://learn.microsoft.com/en-us/visualstudio/version-control/git-manage-repository?view=vs-2022)

---

## ✅ Checklist

* [ ] Azure DevOps remote renamed to `devops`
* [ ] GitHub added as new `origin`
* [ ] Branches & tags pushed to GitHub
* [ ] CI/CD migrated (GitHub Actions or equivalent)
* [ ] Team notified when Azure remote is safe to remove

---

## 🔄 Rollback (if needed)

If migration fails, you can always point back to Azure:

```bash
git remote remove origin
git remote rename devops origin
```

This restores Azure as the default.
