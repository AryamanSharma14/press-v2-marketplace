# Press Integration

## Overview

Press integrates with the marketplace registry in two directions:

1. **Registry → Press:** Press reads `app.toml` files from `frappe/marketplace` and upserts `Marketplace App` doctype records
2. **Press → Telemetry:** Press emits PostHog events when marketplace apps are installed, updated, or removed on customer sites

The registry is authoritative. Press never writes back to it.

## Registry Sync

### Sync job

A scheduled background job in Press syncs the registry every **15 minutes**:

1. Shallow-clones or pulls `frappe/marketplace:main`
2. Reads all `apps/*/app.toml` files
3. For each `app.toml`, upserts the corresponding `Marketplace App` doctype record
4. For each `[[app.versions]]` entry, upserts the corresponding `Marketplace App Version` record (Frappe version → App Source branch mapping)
5. Sets changed records to pending re-validation before going live in the UI

Manual sync is available from the Press admin panel via a "Sync Marketplace Registry" button.

### Version resolution

At install time, Press resolves which branch to use:

1. Get the site's current Frappe version (e.g. `15.18.2`)
2. Find the `[[app.versions]]` entry in the registry where the `frappe` semver range matches
3. Clone `source_url` at the declared `branch`
4. Recursively resolve and install any `requires` dependencies using the same lookup

### Field mapping

| `app.toml` field | `Marketplace App` doctype field |
|-----------------|--------------------------------|
| `app.id` | `app` (Link to App doctype) |
| `app.title` | `title` |
| `app.description` | `description` |
| `app.publisher` | `publisher` ⚠️ new field |
| `app.publisher_name` | `publisher_name` ⚠️ new field |
| `app.license` | `license` ⚠️ new field |
| `app.source_url` | `source_url` ⚠️ new field |
| `app.website` | `website` |
| `app.category` | `categories` |
| `app.tags` | `tags` ⚠️ new field |
| `app.subscription_type` | `subscription_type` |
| `app.pricing_url` | `pricing_url` ⚠️ new field |
| `app.contact.email` | `support` |
| `app.contact.docs_url` | `documentation` |
| `app.contact.issues_url` | `issues_url` ⚠️ new field |
| `app.media.icon` | `image` |
| `app.media.screenshots` | `screenshots` |
| `[[app.versions]]` entries | `Marketplace App Version` child table (one row per entry) |

> Fields marked ⚠️ need to be added to the `Marketplace App` doctype. The current doctype tracks publisher via the `team` Link field — `publisher` and `publisher_name` are new flat fields for the registry-native flow.

### Conflict resolution

If `app.toml` is valid and passes schema check, the Press record is overwritten unconditionally. If an `apps/<id>/` directory is removed from the registry (merged removal PR), the corresponding `Marketplace App` is set to `status = Disabled`. It is not deleted — subscription and billing history must be preserved.

## Telemetry Emission

Press knows exactly when apps change state on customer sites because it manages all deployments. The existing `marketplace_app_hook` in `press/press/doctype/marketplace_app/marketplace_app.py` is called on install and uninstall — PostHog emission is added here.

### Trigger points

| Press deployment event | PostHog event emitted |
|-----------------------|----------------------|
| App added to site and deploy succeeds | `marketplace_app_installed` |
| App removed from site and deploy succeeds | `marketplace_app_uninstalled` |
| App version (release hash) changes on site | `marketplace_app_updated` |

Only apps with a corresponding `Marketplace App` doctype record emit telemetry. Non-marketplace apps are not tracked.

## Full Data Flow

```
frappe/marketplace:main
        │
        │  git pull (every 15 min)
        ▼
  Press sync job
        │
        │  parse app.toml → upsert Marketplace App + version entries
        ▼
  Press DB (MariaDB)
        │
        │  serve to Frappe Cloud UI
        ▼
  Marketplace browse and app detail pages

──────────────────────────────────────────

  Customer installs / removes / updates app
        │
        ▼
  Press deployment flow
        │
        │  is this a Marketplace App? → emit PostHog event
        ▼
  PostHog
        │
        │  query via PostHog API (publisher-filtered)
        ▼
  Developer analytics dashboard
```

## Open Questions

1. **Migration of existing apps** — Existing marketplace listings live in Press DB with no registry entry. Options:
   - Bulk one-time migration PR: generate `app.toml` files for all existing apps
   - Grandfather flag: `legacy = true` on existing `Marketplace App` records until publishers opt in
   - Hard cutover deadline: all existing apps submit registry PRs by a set date

2. **Private source repos** — Can Paid apps keep `source_url` private? Recommended: **yes** — the field is required, but the repo can be private. Press needs a deploy key or GitHub App installation with read access.

3. **App removal notification** — When an app is disabled, should active subscribers receive an email? Recommended: **yes**.

4. **Dependency conflict resolution** — If two apps declare conflicting `requires` constraints for the same dependency, Press should surface this as an error at install time rather than silently installing an incompatible version.
