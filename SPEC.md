# Frappe Marketplace — Specification

## What Is Wrong With The Current System

| Problem | Current behaviour |
|---------|-------------------|
| No history | Marketplace listings live in MariaDB with no audit log |
| Manual publishing | Developers fill a web form on Frappe Cloud |
| No validation | Frappe team manually checks every submission |
| No versioning | A single `branch` field — no awareness of Frappe version compatibility |
| No dependencies | Apps cannot declare what other apps they need |
| Two separate lists | Press has its own DB; bench-cli has `registry/apps.json` — they diverge |
| No CLI install | `bench marketplace install hrms` does not exist |
| No developer analytics | App developers cannot see adoption data |

---

## System Overview

Four components. One data flow.

```
Developer opens PR to frappe/marketplace
        │
        ▼
CI validates app.toml (schema, source_url, semver ranges)
        │
        ▼
Frappe team reviews and merges
        │
        ├──────────────────────────────────┐
        │                                  │
        ▼ (every 15 min)                   ▼ (every hour, or on demand)
Press sync job                      bench-cli registry cache
parse app.toml →                    git pull → ~/.bench-cli/marketplace/
upsert Marketplace App records
        │                                  │
        ▼                                  ▼
Frappe Cloud UI                     bench-cli admin UI (port 8002)
marketplace browse + install        marketplace tab + bench marketplace CLI
        │                                  │
        └──────────────┬───────────────────┘
                       │
                       ▼
              App installed on site
                       │
                       ▼
           Press emits PostHog event
                       │
                       ▼
           Developer analytics dashboard
```

---

## Component 1 — `frappe/marketplace` Registry Repo

This is a new GitHub repository: `https://github.com/frappe/marketplace`

It is the **single source of truth** for all marketplace app listings. It contains no app code — only metadata files.

### Directory Structure

```
frappe/marketplace/
├── apps/
│   ├── frappe-erpnext/
│   │   └── app.toml
│   ├── frappe-hrms/
│   │   └── app.toml
│   └── acme-crm/
│       └── app.toml
├── schema/
│   └── app.schema.json       # JSON Schema — CI validates all app.toml files against this
├── .github/
│   ├── workflows/
│   │   └── validate.yml
│   └── PULL_REQUEST_TEMPLATE.md
└── docs/
```

### Naming Convention

App directories follow `<publisher>-<appname>`:

- `publisher` — GitHub org or username of the app owner. Lowercase, hyphens only.
- `appname` — short identifier. Lowercase, hyphens only.

The `publisher` field inside `app.toml` must match the directory prefix. CI enforces this to prevent squatting.

Examples: `frappe-erpnext`, `frappe-hrms`, `acme-crm`

### `app.toml` Schema

```toml
# ── Identity ──────────────────────────────────────────────────────────────
[app]
id = "hrms"                          # unique identifier, matches <appname> in directory
title = "Frappe HR"                  # display name, max 60 chars
description = "..."                  # plain text, 50–500 chars
publisher = "frappe"                 # GitHub org/username, matches <publisher> in directory
publisher_name = "Frappe Technologies"
license = "MIT"                      # SPDX identifier or "Proprietary"
source_url = "https://github.com/frappe/hrms"   # must be publicly accessible for Free apps
website = "https://frappe.io/products/hrms"     # optional

# ── Categorization ────────────────────────────────────────────────────────
category = "HR & Payroll"   # must be one of the defined categories (see below)
tags = ["payroll", "leaves", "attendance"]      # optional, max 10

# ── Pricing ───────────────────────────────────────────────────────────────
subscription_type = "Free"           # Free | Paid | Freemium
# pricing_url = "https://..."        # required for Paid and Freemium apps

# ── Contact ───────────────────────────────────────────────────────────────
[app.contact]
email = "support@frappe.io"
docs_url = "https://docs.frappe.io/hrms"        # optional
issues_url = "https://github.com/frappe/hrms/issues"  # optional

# ── Media ─────────────────────────────────────────────────────────────────
[app.media]
icon = "hrms/public/images/hrms-logo.svg"       # relative path in app repo, min 256×256
screenshots = [                                 # optional, PNG only, max 8
  "hrms/public/images/screenshots/dashboard.png",
]

# ── Version Matrix ────────────────────────────────────────────────────────
# One entry per supported Frappe version range.
# frappe   = semver range for the Frappe Framework version
# branch   = branch to clone from source_url for this range
# requires = other marketplace apps required at this version (optional)

[[app.versions]]
frappe = ">=14.0.0,<15.0.0"
branch = "version-14"

[[app.versions]]
frappe = ">=15.0.0,<16.0.0"
branch = "version-15"
requires = ["erpnext>=15.0.0,<16.0.0"]

[[app.versions]]
frappe = ">=16.0.0-dev"
branch = "develop"
requires = ["erpnext>=16.0.0-dev"]
```

At least one `[[app.versions]]` entry is required. For apps that do not maintain version branches:

```toml
[[app.versions]]
frappe = ">=15.0.0"
branch = "main"
```

### Valid Categories

```
Accounting & Finance
HR & Payroll
CRM & Sales
Manufacturing
Healthcare
Education
E-commerce
Project Management
Inventory & Supply Chain
Communication
Analytics & Reporting
Developer Tools
Integration
Other
```

### CI Validation (`validate.yml`)

Every PR triggers the validation workflow. It checks:

1. Which `app.toml` files changed in the PR
2. Schema validation against `schema/app.schema.json`
3. `source_url` is publicly reachable (HTTP GET)
4. `publisher` in `app.toml` matches the directory `<publisher>` prefix
5. All semver ranges in `[[app.versions]]` are well-formed PEP 440
6. All `requires` entries reference valid app IDs in the registry
7. Declared `branch` values exist in the source repository

CI must pass before Frappe team review begins.

---

## Component 2 — Press Integration

### Registry Sync Job

A scheduled background job in Press syncs the registry every **15 minutes**.

**Concurrency:** Press runs multiple background workers. Only one sync may run at a time. The job acquires an exclusive file lock on `{clone_dir}/.sync.lock` before touching the working tree. If the lock is held, the incoming run exits immediately — it will be retried on the next 15-minute tick. This prevents two workers from pulling into the same directory simultaneously.

**Steps:**

1. Acquire file lock on `{clone_dir}/.sync.lock`
2. Shallow-clone or pull `frappe/marketplace:main` into a fixed directory on the Press server: `/home/frappe/marketplace-registry/` (created on first run)
3. Read all `apps/*/app.toml` files
4. For each `app.toml`:
   a. If no `App` doctype record exists for `app.id`, create one with `name = app.id` and `repo = source_url` before upserting `Marketplace App`
   b. Upsert the `Marketplace App` doctype record
   c. Upsert each `[[app.versions]]` entry as a `Marketplace App Version` child record
5. If an `apps/<id>/` directory is removed from the registry (merged removal PR), set `Marketplace App.status = Disabled` — do **not** delete; billing and subscription history must be preserved
6. Release file lock

Manual sync available from Press admin panel via "Sync Marketplace Registry" button.

### Doctype Field Mapping

| `app.toml` field | `Marketplace App` doctype field | Status |
|-----------------|---------------------------------|--------|
| `app.id` | `app` (Link → App doctype) — created by sync job if missing | existing |
| `app.title` | `title` | existing |
| `app.description` | `description` | existing |
| `app.publisher` | `publisher` | **new field** |
| `app.publisher_name` | `publisher_name` | **new field** |
| `app.license` | `license` | **new field** |
| `app.source_url` | `source_url` | **new field** |
| `app.website` | `website` | existing |
| `app.category` | `categories` | existing |
| `app.tags` | `tags` | **new field** |
| `app.subscription_type` | `subscription_type` | existing |
| `app.pricing_url` | `pricing_url` | **new field** |
| `app.contact.email` | `support` | existing |
| `app.contact.docs_url` | `documentation` | existing |
| `app.contact.issues_url` | `issues_url` | **new field** |
| `app.media.icon` | `image` | existing |
| `app.media.screenshots` | `screenshots` | existing |
| `[[app.versions]]` entries | `Marketplace App Version` child table | existing (schema update needed) |

The current doctype tracks publisher via a `team` Link field. The new `publisher` and `publisher_name` flat fields are for the registry-native flow and coexist with the existing `team` field during migration.

### Version Resolution at Install Time

When a Frappe Cloud site installs a marketplace app:

1. Get the site's current Frappe version (e.g. `15.18.2`)
2. Find the `[[app.versions]]` entry where the `frappe` semver range matches that version
3. Clone `source_url` at the declared `branch`
4. Recursively resolve and install any `requires` dependencies using the same lookup
5. If two installed apps declare conflicting `requires` constraints — surface this as an error at install time, do not silently install an incompatible version

### Telemetry

Press already integrates PostHog via `press/utils/telemetry.py`. Marketplace telemetry extends this with no new infrastructure.

**Extend the `capture()` signature:**

```python
# Current:
def capture(event, app, distinct_id=None):

# After:
def capture(event, app, distinct_id=None, properties=None):
```

**Events emitted:**

| Press event | PostHog event |
|-------------|--------------|
| App added to site, deploy succeeds | `marketplace_app_installed` |
| App removed from site, deploy succeeds | `marketplace_app_uninstalled` |
| App version hash changes on site | `marketplace_app_updated` |

Only apps with a corresponding `Marketplace App` doctype record emit telemetry.

**Event payload example (`marketplace_app_installed`):**

```json
{
  "event": "marketplace_app_installed",
  "distinct_id": "<site-name>",
  "properties": {
    "app": "hrms",
    "publisher": "frappe",
    "version": "15.0.1",
    "frappe_version": "15",
    "region": "ap-south-1",
    "subscription_type": "Free",
    "$ip": null
  }
}
```

`version` comes from the app's `__version__` in `__init__.py`. `$ip` is always `null`.

### Developer Analytics Dashboard

Available at `/marketplace/publisher/analytics` in Frappe Cloud. All queries server-side filtered by `publisher` — developers only see their own apps.

**Metrics:**
- Total active installs
- Installs over time (weekly, last 12 months)
- Version distribution (% of sites per version)
- Geographic distribution (by AWS region)
- Uninstall rate (signal for app quality)

---

## Component 3 — bench-cli V2 Integration

### Current State

bench-cli V2 (`admin/backend/views/apps.py`) has:

```python
_REGISTRY_PATH = Path(__file__).parent.parent.parent.parent / "registry" / "apps.json"

@apps_bp.route("/registry")
def registry():
    try:
        return jsonify(json.loads(_REGISTRY_PATH.read_text()))
    except Exception:
        return jsonify([])
```

This reads a static JSON file bundled inside the bench-cli repo. The `POST /add` endpoint takes an explicit `branch` parameter — no version resolution.

### What Changes

Three things change in bench-cli:

1. **`GET /registry` reads from local cache instead of the bundled JSON file**
2. **`POST /add` resolves the correct branch from the version matrix**
3. **New `bench marketplace` subcommand group**

### 3a — Registry Cache

bench-cli maintains a local cache of `frappe/marketplace` at `~/.bench-cli/marketplace/`. This path is separate from the old `bench` tool (`~/.bench/`) so both tools can coexist on the same machine without conflict. The cache is global — shared across all benches for the current user.

```
~/.bench-cli/marketplace/
├── apps/
│   ├── frappe-hrms/
│   │   └── app.toml
│   ├── frappe-erpnext/
│   │   └── app.toml
│   └── ...
└── .last-sync              # ISO timestamp of last successful pull
```

**Cache refresh rules:**
- Auto-refresh if `.last-sync` is older than 1 hour
- Force refresh via `bench marketplace sync`
- If cache does not exist, perform initial clone on first access
- If refresh fails (no internet), serve stale cache with a warning logged to the admin UI

**Updating `GET /registry` in `apps.py`:**

```python
CACHE_DIR = Path.home() / ".bench-cli" / "marketplace"

@apps_bp.route("/registry")
def registry():
    _ensure_cache_fresh()
    apps = []
    for toml_path in CACHE_DIR.glob("apps/*/app.toml"):
        try:
            apps.append(_parse_app_toml(toml_path))
        except Exception:
            continue
    return jsonify(apps)
```

The parsed `app.toml` is normalised to the same shape as the old `apps.json` entries so the admin UI frontend requires no changes for basic listing.

### 3b — Version Resolution in `POST /add`

The current endpoint accepts explicit `name`, `repo`, `branch`. With the registry, the branch is resolved automatically from the site's Frappe version.

**Updated flow for marketplace installs:**

```python
@apps_bp.route("/add", methods=["POST"])
def add():
    ...
    # If installing from marketplace (registry_id provided):
    registry_id = data.get("registry_id")
    if registry_id:
        app_meta = _registry_lookup(registry_id)          # reads from cache
        frappe_version = _get_bench_frappe_version(bench_root)
        branch = _resolve_branch(app_meta, frappe_version)  # semver match
        repo = app_meta["source_url"]
        name = app_meta["id"]
    ...
    task_id = TaskRunner(bench_root).run(
        "get-app", {"name": name, "repo": repo, "branch": branch}
    )
```

Direct installs (explicit `repo` + `branch`) continue to work unchanged for non-marketplace apps.

### 3c — `bench marketplace` CLI Commands

New subcommand group added to bench-cli.

#### Consumer Commands

**`bench marketplace search <query>`**
```
bench marketplace search hrms
bench marketplace search --publisher frappe
bench marketplace search --category "HR & Payroll" --frappe-version 15
```
Reads local cache. Auto-refreshes if stale. Output:
```
frappe-hrms      Frappe HR        frappe    Free    v14, v15, v16
acme-crm         Acme CRM         acme      Paid    v15
```

**`bench marketplace install <id>`**
```
bench marketplace install frappe-hrms
bench marketplace install acme-crm --branch main   # override branch
```
1. Looks up `source_url` from registry cache
2. Resolves branch from `[[app.versions]]` semver matrix against current bench Frappe version
3. Calls `TaskRunner("get-app", {name, repo, branch})` — same path as admin UI install

**`bench marketplace list`**
```
App        Registry ID      Installed Commit   Latest Commit   Status
hrms       frappe-hrms      a3f2c91            d8e1b04         outdated
erpnext    frappe-erpnext   9c4f2aa            9c4f2aa         up to date
```

**`bench marketplace update <id>` / `--all`**
Fetches latest cache. For each app, runs `git fetch && git checkout <latest-commit>` in `apps/<name>/`, then reinstalls Python dependencies.

**`bench marketplace sync`**
Force pull of `~/.bench-cli/marketplace/` regardless of cache age.

#### Publisher Commands

**`bench marketplace init`**
```
cd apps/my-app
bench marketplace init
```
Reads `hooks.py` and `setup.py`/`pyproject.toml` to pre-fill known fields. Writes `app.toml` in the current directory. Prints a checklist of fields that need manual review.

**`bench marketplace validate`**
```
cd apps/my-app
bench marketplace validate
```
Runs the same checks as CI, locally:
- All required fields present
- Values match constraints (length, pattern, enum)
- `publisher` matches `source_url` GitHub org
- `media.icon` path exists in the app directory
- (network) `source_url` is reachable

Exit `0` = valid. Exit `1` = errors.

**`bench marketplace publish`**
```
cd apps/my-app
bench marketplace publish
```
1. Runs `bench marketplace validate` — aborts if invalid
2. Checks if `apps/<publisher>-<appname>/` already exists in registry (update vs new listing)
3. Forks `frappe/marketplace` under the publisher's GitHub account via GitHub API
4. Creates branch `add-<publisher>-<appname>` or `bump-<publisher>-<appname>`
5. Commits `app.toml` and pushes
6. Opens PR against `frappe/marketplace:main`
7. Prints PR URL

Requires `GITHUB_TOKEN` env var with `repo` scope, or `gh auth login`.

### 3d — Admin UI Changes

The bench-cli admin UI (Vue frontend, port 8002) marketplace tab already exists. Changes needed:

- **Display app.toml fields**: `publisher_name`, `license`, `subscription_type`, `tags`
- **Version-aware install**: instead of a branch dropdown, show "Compatible with your Frappe version" badge and auto-resolve branch on install
- **Category filter and search**: already exists in bench-cli admin UI; now powered by real registry data
- **`registry_id` passed to `POST /add`** so backend performs version resolution

---

## Component 4 — Publisher Workflow

### Publishing a New App

**Automated (via CLI):**
```bash
cd apps/my-app
bench marketplace init       # scaffold app.toml
bench marketplace validate   # check locally
bench marketplace publish    # open PR
```

**Manual:**
1. Fork `frappe/marketplace`
2. `git checkout -b add-<publisher>-<appname>`
3. `mkdir apps/<publisher>-<appname>` and create `app.toml`
4. Push and open PR against `frappe/marketplace:main`
5. CI runs automatically; fix any failures
6. Frappe team reviews once CI passes

### Updating Metadata or Version Entries

Same flow — edit `app.toml`, open a PR. Any field can be updated this way.

### Review Criteria

| Check | Description |
|-------|-------------|
| CI passing | Schema validates, `source_url` accessible, semver ranges well-formed |
| App installs cleanly | Frappe team tests install on a bench |
| Category appropriate | Category matches actual function |
| Description quality | Accurate, professional |
| Publisher match | `publisher` in `app.toml` matches submitter's GitHub org |
| License accurate | Reflects the actual license |
| Dependencies valid | `requires` entries reference valid registry IDs |

Target: 5 business days for first response on a new listing.

### Removing an App

PR that deletes `apps/<publisher>-<appname>/`. Only original publisher or Frappe team can remove. On merge, Press sets listing to `Disabled` — not deleted (billing history preserved). Active subscribers receive an email notification.

---

## Migration Plan

### Current State

- bench-cli has `registry/apps.json` with ~160 apps. Flat JSON, no versioning, no dependencies.
- Press has `Marketplace App` doctypes in MariaDB. Published via web form.

### Migration Steps

**Phase 1 — Create `frappe/marketplace` repo and migrate existing listings**

1. Create `frappe/marketplace` with directory structure and `schema/app.schema.json`
2. Write a one-time migration script that reads `registry/apps.json` and generates an `app.toml` per app:
   - `id` = `name` from JSON
   - `title`, `description`, `website`, `documentation` from JSON
   - `source_url` = `repo` from JSON
   - `[[app.versions]]` entries generated from `branches[]` array using this branch-name → semver mapping:

     | Branch pattern | Inferred semver range |
     |---|---|
     | `version-14` | `>=14.0.0,<15.0.0` |
     | `version-15` | `>=15.0.0,<16.0.0` |
     | `version-16` | `>=16.0.0,<17.0.0` |
     | `develop` | `>=16.0.0-dev` |
     | `main` / `master` | **flagged** — left as `# REVIEW: set frappe range` |
     | anything else | **flagged** — left as `# REVIEW: set frappe range` |

   - Fields with no JSON equivalent (`license`, `publisher_name`, `tags`, `contact.email`, `subscription_type`) defaulted to safe placeholder values with a `# REVIEW` comment so publishers know what to fill in
   - Any app with one or more flagged entries is added to a `needs-review.txt` list output by the script

3. Submit all generated `app.toml` files in a single bootstrap PR
4. Publishers with flagged entries are notified individually to complete their `app.toml` before the cutover deadline

**Phase 2 — Migrate Press DB listings**

1. Run the same migration script against Press `Marketplace App` records
2. Add the new doctype fields (`publisher`, `publisher_name`, `license`, `source_url`, `tags`, `pricing_url`, `issues_url`)
3. Add `legacy = 1` flag to existing records that have not yet been submitted via registry PR
4. Press sync job goes live — new and updated listings come from the registry; legacy listings remain until publishers opt in

**Phase 3 — Migrate bench-cli**

1. Switch `GET /registry` to read from `~/.bench-cli/marketplace/` cache
2. Deploy updated `apps.py` with version resolution in `POST /add`
3. Remove `registry/apps.json` from bench-cli repo (it is now served from the central registry)
4. Release `bench marketplace` subcommand group

**Phase 4 — Full cutover**

1. Set a deadline for all `legacy = 1` Press listings to submit registry PRs
2. After the deadline, listings that have not moved to the registry are set to `Disabled`
3. Announce migration complete

---

## Open Questions

1. **Private source repos** — Can Paid apps keep `source_url` private? Recommendation: yes — the field is required but the repo can be private. Press needs a deploy key or GitHub App with read access at install time.

2. **Dependency conflict resolution** — If two apps declare conflicting `requires` constraints for the same dependency, the error must be surfaced at install time with a clear message, not silently resolved.

3. **bench-cli V2 Frappe version detection** — `_get_bench_frappe_version()` needs to read the installed Frappe version from `bench_root/apps/frappe/` (e.g. from `frappe/__version__.py`). Implementation detail to confirm during build.

4. **Cache location on multi-user servers** — `~/.bench-cli/marketplace/` is per-user. On a server where multiple users run bench-cli, each user gets their own cache. A system-wide cache at `/var/cache/bench-cli/marketplace/` could be considered for later.

5. **Frappe Cloud ↔ bench-cli parity** — Both should show the same registry. During the migration window, Press may have `legacy` listings not in the registry. The admin UI should indicate when an app is registry-native vs legacy.

6. **App removal notifications** — When Press sets a listing to `Disabled` after a removal PR merges, active subscribers should receive an email. Implementation: existing `marketplace_app_hook` can trigger this.
