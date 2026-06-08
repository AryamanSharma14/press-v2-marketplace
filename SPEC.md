# Frappe Marketplace

**marketplace** is a git-native registry for Frappe apps. It replaces the current database-driven marketplace with a GitHub monorepo where each app is a git submodule, metadata lives in `app.toml` files, and publishing happens via pull requests.

## Core Ideas

- **One repo, all apps.** A central GitHub repo (`frappe/marketplace`) is the source of truth for every listed app
- **Git as version control.** Every change to a listing is a commit. History, diffs, and blame come for free
- **PR-based publishing.** Developers fork, add their app, and open a PR. Frappe team reviews and merges
- **app.toml manifest.** A single TOML file per app describes identity, compatibility, pricing, and media
- **Git submodules for source.** Each app's source code is pinned to a specific commit via a submodule at `apps/<id>/src/`
- **PostHog telemetry.** Press emits events on install/uninstall/update. App developers get GitHub-style usage dashboards
- **bench CLI integration.** Developers can search, install, validate, and publish marketplace apps directly from `bench`

## Quick Start

**Installing an app from the marketplace:**
```bash
bench marketplace install frappe-hrms
```

**Publishing your app:**
```bash
cd apps/my-app
bench marketplace init        # scaffolds app.toml
bench marketplace validate    # checks app.toml locally
bench marketplace publish     # opens PR on frappe/marketplace
```

## Guiding Constraints

1. **Git is authoritative** — the registry repo is the source of truth, never the Press database
2. **No new infra** — telemetry uses existing PostHog integration in Press
3. **Backward compatible** — existing marketplace apps continue working during migration
4. **Fail loudly** — CI validates `app.toml` up-front with actionable errors before Frappe team review
5. **Idempotent sync** — Press syncs registry every 15 minutes; running it twice is always safe

## Repository Structure

```
frappe/marketplace/
├── SPEC.md
├── .gitmodules
├── apps/
│   ├── frappe-erpnext/
│   │   ├── app.toml
│   │   └── src/              ← git submodule → frappe/erpnext
│   └── acme-crm/
│       ├── app.toml
│       └── src/              ← git submodule → acme/crm
├── schema/
│   └── app.schema.json
├── .github/
│   ├── workflows/
│   │   └── validate.yml
│   └── PULL_REQUEST_TEMPLATE.md
└── docs/
```

## Docs

| File | Description |
|------|-------------|
| [docs/architecture.md](docs/architecture.md) | Registry structure, naming conventions, data flow |
| [docs/manifest.md](docs/manifest.md) | app.toml schema and field reference |
| [docs/commands.md](docs/commands.md) | bench marketplace CLI commands |
| [docs/workflow.md](docs/workflow.md) | PR submission and review workflow |
| [docs/telemetry.md](docs/telemetry.md) | Event schema, PostHog integration, developer dashboard |
| [docs/press-sync.md](docs/press-sync.md) | How Press syncs the registry into its database |
