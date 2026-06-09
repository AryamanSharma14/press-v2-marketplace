# PR Submission Workflow

## Overview

Publishing or updating an app on the Frappe Marketplace is a pull request to `frappe/marketplace`. The Frappe team reviews and merges accepted submissions.

Use `bench marketplace publish` to automate this entirely, or follow the manual steps below.

## Publishing a New App (Manual)

**Prerequisites:**
- A public GitHub repository containing a valid Frappe app
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

**3. Add the app directory and app.toml**

```bash
mkdir apps/<publisher>-<appname>
# create app.toml — see docs/manifest.md for the full schema and minimal example
```

**4. Commit and push**

```bash
git add apps/<publisher>-<appname>/
git commit -m "feat(apps): add <publisher>-<appname>"
git push origin add-<publisher>-<appname>
```

**5. Open PR against `frappe/marketplace:main`**

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

## Adding or Updating a Version Entry

To add support for a new Frappe version, or update the branch for an existing one:

```bash
git checkout -b bump-<publisher>-<appname>
# edit apps/<publisher>-<appname>/app.toml
# add or update the relevant [[app.versions]] entry
git add apps/<publisher>-<appname>/app.toml
git commit -m "chore(apps): add v16 support for <publisher>-<appname>"
git push origin bump-<publisher>-<appname>
# open PR
```

## Removing an App

Open a PR that deletes `apps/<publisher>-<appname>/`. Include a reason in the PR description. Only the original publisher or the Frappe team can remove an app.

When merged, Press sets the listing to `Disabled` (not deleted) to preserve subscription and billing history.

## Review Criteria

| Check | Description |
|-------|-------------|
| CI passing | Schema validates, `source_url` is accessible, semver ranges are well-formed |
| App installs cleanly | Frappe team tests install on a bench |
| Category appropriate | Category matches the app's actual function |
| Description quality | Accurate, professional, no spam |
| Publisher match | `publisher` in `app.toml` matches submitter's GitHub org |
| License accurate | License field reflects the actual license |
| Dependencies correct | `requires` entries reference valid app IDs in the registry |

## Review Timeline

Target: **5 business days** for first response. PRs with no response to feedback after 30 days may be closed.

## PR Checklist

`.github/PULL_REQUEST_TEMPLATE.md` is shown automatically when opening a PR:

```markdown
## Checklist

- [ ] app.toml has all required fields
- [ ] source_url points to a public repository
- [ ] At least one [[app.versions]] entry declared
- [ ] Branch names in [[app.versions]] exist in the source repo
- [ ] App installs on the declared Frappe version
- [ ] At least one screenshot included
- [ ] Contact email is actively monitored

## Type

- [ ] New listing
- [ ] Metadata update
- [ ] Version entry added or updated
- [ ] Removal

## Notes for reviewers
```

## Common CI Failures

| Error | Fix |
|-------|-----|
| `Missing required field: app.id` | Add `id` to `[app]` section |
| `publisher does not match directory prefix` | `app.publisher` must match `<publisher>` in directory name |
| `invalid category` | Use one of the categories listed in `docs/manifest.md` |
| `source_url not accessible` | Ensure source repository is public |
| `description too short` | Description must be at least 50 characters |
| `no [[app.versions]] entries` | At least one version entry is required |
| `invalid semver range` | Use PEP 440 format, e.g. `>=15.0.0,<16.0.0` |
| `branch not found in source repo` | Check branch name exists at `source_url` |
