# App Manifest (app.toml)

Every app in the registry must have an `app.toml` at `apps/<id>/app.toml`.

## Full Schema

```toml
# ── Identity ──────────────────────────────────────────────────────────────────

[app]
# Unique identifier. Must match the directory name's <appname> part.
# Pattern: ^[a-z0-9-]+$
id = "hrms"

# Display name shown in marketplace. Max 60 characters.
title = "Frappe HR"

# Plain text description. No markdown. 50–500 characters.
description = "A comprehensive Human Resources module for Frappe Framework covering payroll, leaves, attendance, and performance management."

# GitHub org or username. Must match directory <publisher> prefix.
publisher = "frappe"

# Human-readable publisher display name.
publisher_name = "Frappe Technologies"

# SPDX license identifier or "Proprietary" for closed-source apps.
license = "MIT"

# Source repository URL. Must be publicly accessible for Free apps.
source_url = "https://github.com/frappe/hrms"

# App homepage or documentation site. Optional.
website = "https://frappe.io/products/hrms"


# ── Categorization ────────────────────────────────────────────────────────────

# One of the defined categories. See categories section below.
category = "HR & Payroll"

# Optional search tags. Max 10.
tags = ["payroll", "attendance", "leaves", "performance"]


# ── Pricing ───────────────────────────────────────────────────────────────────

# Free | Paid | Freemium
subscription_type = "Free"

# Required for Paid and Freemium apps. Omit for Free apps.
# pricing_url = "https://example.com/pricing"


# ── Contact ───────────────────────────────────────────────────────────────────

[app.contact]
email = "support@frappe.io"
docs_url = "https://docs.frappe.io/hrms"
issues_url = "https://github.com/frappe/hrms/issues"


# ── Media ─────────────────────────────────────────────────────────────────────

[app.media]
# Path to icon relative to the app repo root. PNG or SVG. Min 256×256px.
icon = "hrms/public/images/hrms-logo.svg"

# Screenshot paths relative to the app repo root. PNG only. Max 8.
screenshots = [
  "hrms/public/images/screenshots/dashboard.png",
  "hrms/public/images/screenshots/payroll.png",
]


# ── Versions ──────────────────────────────────────────────────────────────────
#
# Each entry answers: "if a site is running Frappe X, which branch do I clone
# and what other apps does this need?"
#
# frappe   — semver range for the Frappe Framework version
# branch   — branch to clone from source_url for this range
# requires — other marketplace apps required at this version (optional)

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

## Versioning

Each app doesn't have a single version — it has a version per Frappe Framework version. ERPNext v15 lives on a different branch from ERPNext v14, and HRMS on v15 only works with ERPNext on v15, not v14. On top of that, some repos like payments have no tags at all, so you can't rely on semver releases.

Each `[[app.versions]]` entry maps a framework semver range to a branch and optional dependencies. At install time Press checks what Frappe version the site is running, finds the matching entry, clones that branch from `source_url`, then does the same recursively for any declared dependencies.

**Well-maintained repo** (version branches, proper tags):
```toml
[[app.versions]]
frappe = ">=15.0.0,<16.0.0"
branch = "version-15"
requires = ["erpnext>=15.0.0,<16.0.0"]
```

**Poorly-maintained repo** (no version branches, no tags):
```toml
[[app.versions]]
frappe = ">=15.0.0"
branch = "main"
```

## Field Reference

### Top-level fields

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `app.id` | string | ✓ | `^[a-z0-9-]+$`, matches directory `<appname>` |
| `app.title` | string | ✓ | Max 60 chars |
| `app.description` | string | ✓ | Plain text, 50–500 chars |
| `app.publisher` | string | ✓ | `^[a-z0-9-]+$`, matches directory `<publisher>` |
| `app.publisher_name` | string | ✓ | — |
| `app.license` | string | ✓ | SPDX identifier or `Proprietary` |
| `app.source_url` | string | ✓ | Public HTTPS URL |
| `app.website` | string | — | HTTPS URL |
| `app.category` | string | ✓ | Must be one of defined categories |
| `app.tags` | string[] | — | Max 10 items |
| `app.subscription_type` | enum | ✓ | `Free`, `Paid`, or `Freemium` |
| `app.pricing_url` | string | Paid/Freemium | HTTPS URL |
| `app.contact.email` | string | ✓ | Valid email |
| `app.contact.docs_url` | string | — | HTTPS URL |
| `app.contact.issues_url` | string | — | HTTPS URL |
| `app.media.icon` | string | ✓ | Relative path in app repo |
| `app.media.screenshots` | string[] | — | Max 8, relative paths in app repo |

### `[[app.versions]]` fields

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `frappe` | string | ✓ | PEP 440 semver range, e.g. `>=15.0.0,<16.0.0` |
| `branch` | string | ✓ | Branch name in `source_url` repo |
| `requires` | string[] | — | App IDs with semver constraints, e.g. `erpnext>=15.0.0,<16.0.0` |

At least one `[[app.versions]]` entry is required.

## Categories

`app.category` must be exactly one of:

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

## Minimal Valid Example

```toml
[app]
id = "my-app"
title = "My App"
description = "A Frappe app that does something useful for small businesses managing their inventory."
publisher = "mycompany"
publisher_name = "My Company Ltd"
license = "MIT"
source_url = "https://github.com/mycompany/my-app"
category = "Inventory & Supply Chain"
subscription_type = "Free"

[app.contact]
email = "hello@mycompany.com"

[app.media]
icon = "my_app/public/images/icon.png"

[[app.versions]]
frappe = ">=15.0.0,<16.0.0"
branch = "version-15"
```
