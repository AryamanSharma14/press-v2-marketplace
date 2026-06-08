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
4. Stores the submodule commit hash as `registry_version` on the record (new field — must be added to the `Marketplace App` doctype)
5. Sets changed records to pending re-validation before going live in the UI

Manual sync is available from the Press admin panel via a "Sync Marketplace Registry" button.

### Field mapping

| `app.toml` field | `Marketplace App` doctype field |
|-----------------|--------------------------------|
| `app.id` | `app` (Link to App doctype) |
| `app.title` | `title` |
| `app.description` | `description` |
| `app.publisher` | `publisher` ⚠️ new field — does not exist yet on doctype |
| `app.publisher_name` | `publisher_name` ⚠️ new field — does not exist yet on doctype |
| `app.license` | `license` ⚠️ new field — does not exist yet on doctype |
| `app.source_url` | `source_url` ⚠️ new field — does not exist yet on doctype |
| `app.website` | `website` |
| `app.category` | `categories` |
| `app.tags` | `tags` ⚠️ new field — does not exist yet on doctype |
| `app.frappe_versions` | `supported_versions` (via `sources` child table) |
| `app.subscription_type` | `subscription_type` |
| `app.pricing_url` | `pricing_url` ⚠️ new field — does not exist yet on doctype |
| `app.contact.email` | `support` (existing support email field) |
| `app.contact.docs_url` | `documentation` (existing documentation field) |
| `app.contact.issues_url` | `issues_url` ⚠️ new field — does not exist yet on doctype |
| `app.media.icon` | `image` |
| `app.media.screenshots` | `screenshots` |
| _(registry commit)_ | `registry_version` ⚠️ new field — does not exist yet on doctype |

> **Note:** Fields marked ⚠️ need to be added to the `Marketplace App` doctype as part of this rework. The current doctype tracks publisher via the `team` Link field (Team doctype) — `publisher` and `publisher_name` are new flat fields proposed for the registry-native flow.

### Conflict resolution

If `app.toml` is valid and passes schema check, the Press record is overwritten unconditionally. If an `apps/<id>/` directory is removed from the registry (merged removal PR), the corresponding `Marketplace App` is set to `status = Unlisted`. It is not deleted — subscription and billing history must be preserved.

## Telemetry Emission

Press knows exactly when apps change state on customer sites because it manages all deployments.

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
        │  parse app.toml → upsert Marketplace App records
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

These require a decision from the Frappe team before implementation:

1. **Submodule depth during sync** — Should the sync job run `git submodule update` or just read the commit hash from `.gitmodules`? Recommended: **hash only** — Press does not need app source code, only the metadata in `app.toml`.

2. **Migration of existing apps** — Existing marketplace listings live in Press DB with no registry entry. Options:
   - Bulk one-time migration PR: generate `app.toml` files for all existing apps
   - Grandfather flag: `legacy = true` on existing `Marketplace App` records until publishers opt in
   - Hard cutover deadline: all existing apps submit registry PRs by a set date

3. **Private source repos** — Can Paid apps keep `source_url` private? Recommended: **yes** — the field is required, but the repo can be private. The sync job needs a deploy key or GitHub App installation with read access.

4. **App removal notification** — When an app is unlisted, should active subscribers receive an email? Recommended: **yes**.
