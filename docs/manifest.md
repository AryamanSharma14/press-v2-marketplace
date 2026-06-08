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


# ── Compatibility ─────────────────────────────────────────────────────────────

# Supported Frappe Framework major versions. At least one required.
frappe_versions = ["14", "15"]


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
# Path to icon relative to src/ submodule root. PNG or SVG. Min 256×256px.
icon = "hrms/public/images/hrms-logo.svg"

# Screenshot paths relative to src/ submodule root. PNG only. Max 8.
screenshots = [
  "hrms/public/images/screenshots/dashboard.png",
  "hrms/public/images/screenshots/payroll.png",
]
```

## Field Reference

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
| `app.frappe_versions` | string[] | ✓ | Non-empty, values like `"14"`, `"15"` |
| `app.subscription_type` | enum | ✓ | `Free`, `Paid`, or `Freemium` |
| `app.pricing_url` | string | Paid/Freemium | HTTPS URL |
| `app.contact.email` | string | ✓ | Valid email |
| `app.contact.docs_url` | string | — | HTTPS URL |
| `app.contact.issues_url` | string | — | HTTPS URL |
| `app.media.icon` | string | ✓ | Relative path in `src/` |
| `app.media.screenshots` | string[] | — | Max 8, relative paths in `src/` |

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
frappe_versions = ["15"]
subscription_type = "Free"

[app.contact]
email = "hello@mycompany.com"

[app.media]
icon = "my_app/public/images/icon.png"
```
