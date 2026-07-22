---
name: github-gitlab-mirror
description: >-
  Set up and troubleshoot GitHub↔GitLab repository mirroring (pull or push),
  including PAT scopes, workflow scope errors, diverged main, and Mintlify
  cutover order. Use when migrating hosting to GitLab, configuring a mirror,
  syncing GitHub and GitLab mains, or the user mentions mirror, pull mirror,
  push mirror, or invalid credentials for GitLab mirroring.
---

# GitHub ↔ GitLab repository mirroring

## Decide direction first

| Goal | Mirror direction | Primary for merges |
|------|------------------|--------------------|
| GitHub stays source of truth (e.g. Mintlify still on GitHub) | GitLab **Pull** from GitHub | **GitHub** |
| GitLab is primary (post-cutover) | GitLab **Push** to GitHub | **GitLab** |

Never run active Pull and Push mirrors against each other on the same pair of repos.

## Initial seed (empty GitLab project)

1. Create empty GitLab project (gitlab.com or self-hosted; Mintlify needs public reachability).
2. Mirror-clone and push (avoid leaving `refs/pull/*` if possible):

```powershell
git clone --mirror https://github.com/ORG/REPO.git tmp-mirror
cd tmp-mirror
git remote set-url --push origin https://gitlab.com/GROUP/PROJECT.git
git push --mirror
# Optional: unset remote.origin.mirror, then delete refs/pull/* from GitLab
```

3. Local remotes after cutover planning:
   - `origin` → GitLab
   - `github` → GitHub

## Auth for GitLab mirroring (HTTPS)

GitHub **account passwords do not work**. Use a **classic PAT**.

| Field | Value |
|-------|--------|
| Repo URL | `https://github.com/ORG/REPO.git` |
| Username | GitHub username (not email) |
| Password | Classic PAT |

**Classic PAT scopes:**
- Pull mirror: `repo`
- Push mirror that updates `.github/workflows/`: `repo` **and** `workflow`

Fine-grained tokens often only list personal repos unless the org enables them and **Resource owner** is the org. Prefer classic for org repos.

**SAML SSO:** If the org uses SAML, open https://github.com/settings/tokens → token → **Configure SSO** → **Authorize** for the org. If Configure SSO is missing, SSO may not apply; verify with `git ls-remote` instead.

**Prove token works before blaming GitLab:**

```powershell
git ls-remote "https://USERNAME:PAT@github.com/ORG/REPO.git" HEAD
```

## Keep mains aligned before enabling Pull

If GitLab is ahead of GitHub (common after GitLab-only CI merges):

```powershell
git fetch github
git fetch origin
git log --oneline github/main..origin/main   # only on GitLab
git log --oneline origin/main..github/main   # only on GitHub
```

- GitLab ahead only (GitHub is ancestor): `git push github main` (needs `workflow` if workflows changed).
- True diverge (commits on both): pick a winner; force-align the loser (`git reset --hard` + `git push --force` on that remote only after explicit user OK).

Pull mirror error *"default branch has diverged"* = branches not fast-forward compatible. Sync first, then Update now.

## Configure Pull mirror (GitHub primary)

1. Sync so both `main` SHAs match.
2. GitLab → **Settings → Repository → Mirroring repositories**
3. Add mirror: URL above, direction **Pull**, username + PAT
4. Remove/disable conflicting **Push** mirrors
5. **Update now** → confirm success

**Operating rules while Pull is active:**
- Merge on **GitHub** only
- Do not merge long-lived work only on GitLab (next pull can overwrite)
- CI variables, MRs/PRs, schedules are **not** mirrored

## Configure Push mirror (GitLab primary)

Same UI, direction **Push**, PAT with `repo` + `workflow`. Merge on GitLab; GitHub follows.

## Common errors

| Error | Fix |
|-------|-----|
| `invalid credentials` / `Authentication failed` | Bad/stale PAT; re-paste into GitLab; authorize SSO; use classic PAT |
| `workflow` scope refused on push | Add `workflow` to classic PAT |
| `default branch has diverged` | Align mains (see above), then retry |
| Fine-grained token: only personal repos | Use classic PAT or set Resource owner to the org |

## Mintlify + mirror order

1. Keep Mintlify on GitHub while Pull mirror is active.
2. When going live on GitLab: reconnect Mintlify to GitLab (project ID, token `api`+`read_api`, webhook `https://leaves.mintlify.com/gitlab-webhook`).
3. Then flip mirror to **Push** (GitLab → GitHub) so GitLab is primary.
4. Disconnect old Mintlify GitHub connection to avoid dual deploys.

## CI note

Mirroring copies git history only. Port GitHub Actions → `.gitlab-ci.yml` separately; recreate CI/CD variables and schedules on GitLab.
