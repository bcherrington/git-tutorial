# Step-by-Step: Migrating Azure Repos → GitHub using GitHub Importer

## 1. Prep Work

1. **Check your Azure DevOps repo URL**

   * Go to Azure DevOps → Repos → *Clone* button → copy the HTTPS clone URL.
     Example:

   ```
   https://dev.azure.com/<org>/<project>/_git/<repo>
   ```

2. **Personal Access Token (PAT)** for Azure DevOps

   * In Azure DevOps, click your profile → **Personal access tokens**.
   * New Token → scopes: **Code: (read)**.
   * Copy it (you’ll paste into GitHub Importer).

3. **GitHub destination**

   * Decide the GitHub **organization** (or your account) where the repo will live.
   * Ensure you have permission to create repositories there.
   * Decide visibility: private vs public.

---

## 2. Start the Import

1. In GitHub, go to:
   [https://github.com/new/import](https://github.com/new/import)

2. Fill in:

   * **Your old repository’s clone URL**: paste the Azure DevOps repo URL.
   * **Your old repository’s login**: Azure DevOps username/email.
   * **Your old repository’s password**: paste the Azure DevOps PAT.

3. Click **Begin import**.

---

## 3. Configure Destination Repo

* Enter the **new repository name** (default is same as Azure).
* Choose **Owner** (your GitHub org or user).
* Choose **Visibility**: Private (default recommended) or Public.
* Click **Create repository**.

---

## 4. Wait & Verify

* GitHub will show progress as it imports commits, branches, and tags.
* For medium repos this takes minutes; large repos may take longer.
* Once complete, you’ll be redirected to the new GitHub repo page.

---

## 5. Post-Migration Steps

1. **Set default branch** to match your workflow (`main` as per the Git Best Practices SOP in this repository).
2. **Protect branches** (Settings → Branches):

   * Require pull requests before merging.
   * Require at least 1 reviewer.
   * Restrict merge strategies to **Squash merge** (as recommended in the Git Best Practices SOP).
3. **Add PR template & CODEOWNERS** in `.github/`.
4. **Reconnect Jira**:

   * Install **GitHub for Jira** app from Atlassian Marketplace.
   * Verify Jira keys in commit titles and PRs are linked.
5. **Update local clones**:

   ```bash
   git remote set-url origin https://github.com/<org>/<repo>.git
   git fetch origin
   ```
6. **Freeze old repo**: mark Azure DevOps repo read-only or remove write permissions.

---

## 6. What’s Not Imported

* Pull requests & review comments.
* Azure branch policies (recreate them in GitHub).
* Pipelines (if any exist, they will need to be recreated in GitHub Actions).
* Work items/issues (these remain in Azure DevOps and need separate migration planning).

---

✅ At this point, your repo is fully on GitHub, with all history and branches intact, and your team can switch their `origin` remotes.

