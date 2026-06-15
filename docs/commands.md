# bench marketplace Commands

`bench marketplace` is a subcommand group to be added to bench-cli for interacting with the Frappe Marketplace registry. It covers two workflows: **consuming** apps (search, install, update) and **publishing** apps (init, validate, publish).

> **Status:** Not yet implemented. Current bench-cli (v5.x) has no `marketplace` command. These are proposed commands for the registry rework.

## Consumer Commands

### `bench marketplace search <query>`

Search the local registry cache for apps matching a name, publisher, or tag.

```bash
bench marketplace search hrms
bench marketplace search "project management"
bench marketplace search --publisher frappe
```

**Options:**
- `--publisher <name>` — filter by publisher
- `--category <name>` — filter by category
- `--frappe-version <version>` — filter by Frappe compatibility

**Output:**
```
frappe-hrms         Frappe HR             frappe      Free    v14, v15
acme-crm            Acme CRM              acme        Paid    v15
```

Reads from the local cache at `~/.bench-cli/marketplace/`. Auto-refreshes if cache is older than 1 hour.

---

### `bench marketplace install <id>`

Install a marketplace app into the current bench. `<id>` is the registry directory name (`<publisher>-<appname>`).

```bash
bench marketplace install frappe-hrms
bench marketplace install acme-crm --branch main
```

**What it does:**
1. Looks up `source_url` from the registry cache for `<id>`
2. Runs `bench get-app <source_url>` to clone and install
3. Records the registry `id` in the bench's app metadata

**Options:**
- `--branch <branch>` — override the default branch

**Exit codes:**
- `0` — installed successfully
- `1` — app not found in registry
- `2` — `bench get-app` failed

---

### `bench marketplace list`

List all marketplace apps installed in the current bench, with their registry versions.

```bash
bench marketplace list
```

**Output:**
```
App             Registry ID       Installed Commit   Latest Commit   Status
hrms            frappe-hrms       a3f2c91            d8e1b04         outdated
erpnext         frappe-erpnext    9c4f2aa            9c4f2aa         up to date
```

---

### `bench marketplace update <id>`

Update an installed marketplace app to the commit pinned in the current registry.

```bash
bench marketplace update frappe-hrms
bench marketplace update --all
```

**Options:**
- `--all` — update all installed marketplace apps

**What it does:**
1. Fetches the latest registry cache (refreshes `~/.bench-cli/marketplace/`)
2. Resolves the correct branch for `<id>` from the updated cache
3. Runs `git pull origin <branch>` in the app directory to advance to the latest commit on that branch
4. Runs `pip install -e apps/<app>` to reinstall dependencies

---

### `bench marketplace sync`

Force a refresh of the local registry cache, regardless of cache age.

```bash
bench marketplace sync
```

Performs a shallow `git pull` on `~/.bench-cli/marketplace/`.

---

## Publisher Commands

### `bench marketplace init`

Scaffold an `app.toml` for the Frappe app in the current directory. Reads `hooks.py` and `setup.py` to pre-fill known fields.

```bash
cd apps/my-app
bench marketplace init
```

**What it does:**
1. Reads `my_app/hooks.py` for `app_name`, `app_title`, `app_description`, `app_publisher`, `app_email`
2. Reads `setup.py` or `pyproject.toml` for `install_requires`
3. Writes a pre-filled `app.toml` in the current directory
4. Prints a checklist of fields that need manual review

**Output:**
```
Created app.toml — review these fields before publishing:
  [ ] category       — set to "Other", update to correct category
  [ ] frappe_versions — set to ["15"], confirm supported versions
  [ ] media.icon     — no icon found, add path to your app icon
  [ ] contact.docs_url — not set
```

---

### `bench marketplace validate`

Validate `app.toml` in the current directory against the registry schema. Runs the same checks as CI.

```bash
cd apps/my-app
bench marketplace validate
```

**Checks performed (local, no network required):**
1. All required fields present
2. Field values match constraints (length, pattern, enum)
3. `publisher` in `app.toml` matches the `<publisher>` component of `source_url`
4. `media.icon` path exists in the app directory

**Checks performed (requires network, skipped offline):**
5. `source_url` is a reachable public URL

**Output on success:**
```
app.toml is valid.
```

**Output on failure:**
```
app.toml validation failed:
  [error] app.category: "Misc" is not a valid category
  [error] app.media.icon: path "my_app/public/icon.png" does not exist
  [warn]  app.contact.docs_url: not set (recommended)
```

**Exit codes:**
- `0` — valid
- `1` — validation errors

---

### `bench marketplace publish`

Open a pull request on `frappe/marketplace` to list or update the current app.

```bash
cd apps/my-app
bench marketplace publish
```

**What it does:**
1. Runs `bench marketplace validate` — aborts if invalid
2. Checks if an entry for this app already exists in the registry (update vs new listing)
3. Forks `frappe/marketplace` under the publisher's GitHub account (via GitHub API)
4. Creates a branch `add-<publisher>-<appname>` or `bump-<publisher>-<appname>`
5. Adds `apps/<publisher>-<appname>/app.toml` to the branch (new listing) or updates the existing `app.toml` (metadata/version update)
6. Commits and pushes
7. Opens a PR against `frappe/marketplace:main`
8. Prints the PR URL

**Requirements:**
- `GITHUB_TOKEN` environment variable with `repo` scope, or interactive GitHub auth via `gh auth login`

**Output:**
```
Validating app.toml... ok
Forking frappe/marketplace... done (polling until ready)
Creating branch add-acme-crm... done
Writing apps/acme-crm/app.toml... done
Pushing... done

PR opened: https://github.com/frappe/marketplace/pull/142
```

---

## Exit Codes

All `bench marketplace` commands follow the same exit code convention:

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | User or configuration error (missing field, app not found) |
| `2` | Unexpected failure (network error, git error) |
