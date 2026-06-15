# Architecture

## Why This Over the Current Marketplace

The current marketplace stores app listings as Frappe doctypes in MariaDB. This works but has real limitations:

| | Current (DB-driven) | New (git-native) |
|--|---------------------|-----------------|
| Change history | None | Full git log — who changed what, when, why |
| Publishing | Web form on Frappe Cloud | GitHub PR |
| Validation | Manual by Frappe team | Automated CI on every PR |
| Versioning | Ad-hoc, per app | Standardized `[[app.versions]]` semver matrix |
| Dependencies | Not declared | Explicit per-version `requires` |
| Fix a typo | Only publisher can, via dashboard | Anyone can open a PR |
| Install via CLI | Not possible | `bench marketplace install <id>` |

The source URL already exists in the current system (`App Source.repository_url`). What changes is everything around it.

## Repository

Central registry: `https://github.com/frappe/marketplace`

This is the source of truth for all Frappe marketplace app listings. It contains **metadata only** — no app code. Press clones directly from each app's `source_url` at the right branch when installing.

## Directory Structure

```
frappe/marketplace/
├── apps/
│   ├── frappe-erpnext/
│   │   └── app.toml          # metadata only
│   ├── frappe-hrms/
│   │   └── app.toml
│   └── acme-crm/
│       └── app.toml
├── schema/
│   └── app.schema.json       # JSON Schema — CI validates all app.toml against this
├── .github/
│   ├── workflows/
│   │   └── validate.yml      # runs on every PR
│   └── PULL_REQUEST_TEMPLATE.md
└── docs/
```

No submodules. Each `app.toml` is self-contained — it declares the source URL and which branch maps to which framework version.

## Naming Convention

App directories follow the pattern `<publisher>-<appname>`:

- `publisher` — GitHub org or username of the app owner. Lowercase, hyphens allowed
- `appname` — short identifier for the app. Lowercase, hyphens allowed

The `publisher` field in `app.toml` must match the directory's `<publisher>` prefix. CI enforces this to prevent name squatting.

**Examples:**
- `frappe-erpnext` — ERPNext by Frappe
- `frappe-hrms` — Frappe HR
- `acme-crm` — CRM app by Acme Corp

## Versioning Model

Each app declares a `[[app.versions]]` table — one entry per supported Frappe Framework version range. Each entry specifies:

- `frappe` — a semver range for the framework (e.g. `>=15.0.0,<16.0.0`)
- `branch` — which branch of the source repo to install for this range
- `requires` — optional list of other apps required at this version (with their own semver constraints)

**Well-maintained repo** (has version branches and tags):
```toml
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

**Poorly maintained repo** (no version branches, just main):
```toml
[[app.versions]]
frappe = ">=15.0.0"
branch = "main"
```

This handles both extremes. When Press installs an app, it looks up which version entry matches the site's current Frappe version and clones the declared branch.

## CI Validation

Every PR triggers `.github/workflows/validate.yml`. The workflow:

1. Detects which `app.toml` files changed in the PR
2. Validates each against `schema/app.schema.json`
3. Checks `source_url` is publicly accessible
4. Verifies `publisher` in `app.toml` matches the directory prefix
5. Validates all semver ranges in `[[app.versions]]` are well-formed
6. Checks declared `requires` reference valid app IDs in the registry

CI must pass before Frappe team review begins.

## Data Flow

```
Developer fork + PR
        │
        ▼
  CI validation (auto)
        │
        ▼
  Frappe team review + merge
        │
        ▼
  frappe/marketplace:main updated
        │
        │  git pull (every 15 min)
        ▼
  Press background sync job
        │
        │  parse app.toml → upsert Marketplace App records
        ▼
  Frappe Cloud UI
  (marketplace browse and app detail pages)
```

## Local Registry Cache (bench CLI)

`bench` maintains a shallow local cache of the registry at `~/.bench-cli/marketplace/`. Used for `bench marketplace search` and `bench marketplace install` without requiring a GitHub API token.

```
~/.bench-cli/marketplace/
├── apps/
│   └── */app.toml
└── .last-sync
```

Cache refreshes automatically when older than 1 hour, or on demand via `bench marketplace sync`.
