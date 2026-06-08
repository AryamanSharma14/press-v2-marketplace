# PR Submission Workflow

## Overview

Publishing or updating an app on the Frappe Marketplace is a pull request to `frappe/marketplace`. The Frappe team reviews and merges accepted submissions.

Use `bench marketplace publish` to automate this entirely, or follow the manual steps below.

## Publishing a New App (Manual)

**Prerequisites:**
- A public GitHub repository containing a valid Frappe app
- At least one tagged release in your repository
- A GitHub account

**1. Fork and clone**

```bash
git clone https://github.com/<your-org>/marketplace
cd marketplace
```

**2. Create a branch**

```bash
git checkout -b add-<publisher>-<appname>
```

**3. Add the app directory and submodule**

```bash
mkdir apps/<publisher>-<appname>
git submodule add https://github.com/<your-org>/<repo> apps/<publisher>-<appname>/src
```

**4. Create app.toml**

Create `apps/<publisher>-<appname>/app.toml` from scratch, filling in all required fields. See [docs/manifest.md](manifest.md) for the full schema and a minimal valid example to copy from.

**5. Commit and push**

```bash
git add apps/<publisher>-<appname>/
git commit -m "feat(apps): add <publisher>-<appname>"
git push origin add-<publisher>-<appname>
```

**6. Open PR against `frappe/marketplace:main`**

CI runs automatically. Fix any failures and push to the same branch. Frappe team reviews once CI passes.

## Updating Metadata

To update description, screenshots, contact info, or any `app.toml` field:

```bash
git checkout -b update-<publisher>-<appname>
# edit apps/<publisher>-<appname>/app.toml
git add apps/<publisher>-<appname>/app.toml
git commit -m "chore(apps): update <publisher>-<appname> metadata"
git push origin update-<publisher>-<appname>
# open PR
```

## Bumping the App Version

To update the listed version to a newer commit in the source repo:

```bash
git checkout -b bump-<publisher>-<appname>
# fetch latest from origin and check out the desired tag or commit
git -C apps/<publisher>-<appname>/src fetch origin
git -C apps/<publisher>-<appname>/src checkout <tag-or-commit>
git add apps/<publisher>-<appname>/src
git commit -m "chore(apps): bump <publisher>-<appname> to $(git -C apps/<publisher>-<appname>/src rev-parse --short HEAD)"
git push origin bump-<publisher>-<appname>
# open PR
```

Note: submodules are checked out in detached HEAD state. Do not use `git pull` on the submodule directory — use `fetch` + `checkout` to a specific tag or commit.

## Removing an App

Open a PR that deletes `apps/<publisher>-<appname>/`. Include a reason in the PR description. Only the original publisher or the Frappe team can remove an app.

When merged, Press sets the listing to `Disabled` (not deleted) to preserve subscription and billing history.

## Review Criteria

| Check | Description |
|-------|-------------|
| CI passing | Schema validates, submodule URL is accessible |
| App installs cleanly | Frappe team tests install on a bench |
| Category appropriate | Category matches the app's actual function |
| Description quality | Accurate, professional, no spam |
| Publisher match | `publisher` in `app.toml` matches submitter's GitHub org |
| License accurate | License field reflects the actual license |

## Review Timeline

Target: **5 business days** for first response. PRs with no response to feedback after 30 days may be closed.

## PR Checklist

`.github/PULL_REQUEST_TEMPLATE.md` is shown automatically when opening a PR:

```markdown
## Checklist

- [ ] app.toml has all required fields
- [ ] Submodule points to a public repository
- [ ] App installs on Frappe v14 or v15
- [ ] At least one screenshot included
- [ ] Contact email is actively monitored

## Type

- [ ] New listing
- [ ] Metadata update
- [ ] Version bump
- [ ] Removal

## Notes for reviewers
```

## Common CI Failures

| Error | Fix |
|-------|-----|
| `Missing required field: app.id` | Add `id` to `[app]` section |
| `publisher does not match directory prefix` | `app.publisher` must match `<publisher>` in directory name |
| `invalid category` | Use one of the categories listed in `docs/manifest.md` |
| `submodule URL not accessible` | Ensure source repository is public |
| `description too short` | Description must be at least 50 characters |
