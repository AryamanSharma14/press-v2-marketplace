# Architecture

## Registry Repository

The marketplace registry lives at `https://github.com/frappe/marketplace`. It is fully public. Every change is a commit, every listing change is reviewable via `git log`, and every app version bump is traceable via `git blame`.

## Directory Structure

```
frappe/marketplace/
├── .gitmodules                      # all submodule declarations
├── apps/
│   ├── frappe-erpnext/
│   │   ├── app.toml                 # metadata (required)
│   │   └── src/                     # submodule → frappe/erpnext@<commit>
│   ├── frappe-hrms/
│   │   ├── app.toml
│   │   └── src/                     # submodule → frappe/hrms@<commit>
│   └── acme-crm/
│       ├── app.toml
│       └── src/                     # submodule → acme/crm@<commit>
├── schema/
│   └── app.schema.json              # JSON Schema — CI validates all app.toml against this
├── .github/
│   ├── workflows/
│   │   └── validate.yml             # runs on every PR
│   └── PULL_REQUEST_TEMPLATE.md
└── docs/
```

## Naming Convention

App directories follow the pattern `<publisher>-<appname>`:

- `publisher` — GitHub org or username of the app owner. Lowercase, hyphens allowed
- `appname` — short identifier for the app. Lowercase, hyphens allowed

The `publisher` field in `app.toml` must match the directory's `<publisher>` prefix. CI enforces this to prevent name squatting.

**Examples:**
- `frappe-erpnext` — ERPNext by Frappe
- `frappe-hrms` — Frappe HR
- `acme-crm` — CRM app by Acme Corp

## Submodule Convention

Each app's source is pinned at `apps/<id>/src/` as a git submodule pointing to a **specific commit**, not a branch. This means:

- The registry records the exact code version that is listed at any point in time
- Updating a listing to a newer release requires a new PR that bumps the submodule commit
- `git log apps/frappe-hrms/src` shows the full version history of that listing

## Versioning

There are no separate version directories. Each entry represents the **current published version**. Historical versions are accessible via `git log` on the registry. This is the same model used by `nixpkgs` and `homebrew-core`.

## CI Validation

Every PR triggers `.github/workflows/validate.yml`. The workflow:

1. Detects which `app.toml` files changed in the PR
2. Validates each against `schema/app.schema.json`
3. Checks the submodule URL is publicly accessible
4. Verifies `publisher` in `app.toml` matches the directory prefix
5. Checks description length and required fields

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

`bench` maintains a shallow local cache of the registry at `~/.bench/marketplace/`. This is used for `bench marketplace search` and `bench marketplace install` without requiring a GitHub API token.

```
~/.bench/marketplace/
├── apps/
│   └── */app.toml    ← metadata only, no submodules
└── .last-sync        ← timestamp of last pull
```

The cache is refreshed automatically when it is older than 1 hour, or on demand via `bench marketplace sync`.
