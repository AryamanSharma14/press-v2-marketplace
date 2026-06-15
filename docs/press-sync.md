# Press Integration

## Overview

Press integrates with the marketplace registry in two directions:

1. **Registry → Press:** Press reads `app.toml` files from `frappe/marketplace` and upserts `Marketplace App` doctype records
2. **Press → Telemetry:** Press emits PostHog events when marketplace apps are installed, updated, or removed on customer sites

The registry is authoritative. Press never writes back to it.

## Registry Sync

### Sync job

A scheduled background job in Press syncs the registry every **hour** (registered in `hooks.py` under `scheduler_events["hourly"]`):

1. Acquires a distributed Redis lock: `frappe.cache().set("marketplace_sync_lock", 1, nx=True, ex=900)`. If the lock is not acquired (another worker on any machine is syncing), exits immediately — retries on the next hourly tick. File-based locks do not work across servers; Redis is correct here because Press already uses Redis for RQ.
2. Shallow-clones or pulls `frappe/marketplace:main` into `/home/frappe/marketplace-registry/` (created on first run).
3. Reads all `apps/*/app.toml` files.
4. For each `app.toml`:
   - If no `App` doctype record exists for `app.id`, creates one (`name = app.id`, `title = app.title`) before upserting `Marketplace App`. Note: `App` autoname is "Prompt" — set the `name` field explicitly in the insert.
   - Loads or creates the `Marketplace App` record and writes all top-level fields.
   - Replaces `Marketplace App Registry Version` child rows entirely: set `doc.registry_versions = []`, then `doc.append(...)` one row per `[[app.versions]]` entry with `frappe_range`, `branch`, and `requires` (stored as JSON string). Call `doc.save(ignore_permissions=True)` — Frappe writes parent and child rows atomically. Do **not** try to upsert child rows by a business key; Frappe child rows have auto-generated `name` fields and offer no such lookup.
5. For each `apps/<id>/` that was present in the previous sync but is now absent (removal PR merged): `frappe.db.set_value("Marketplace App", app_id, "status", "Disabled")`. Per Press conventions (`taste.md`), `frappe.db.set_value` is correct for a single-field update. Do not delete — subscription and billing history must be preserved.
6. Releases Redis lock: `frappe.cache().delete("marketplace_sync_lock")`.

Manual sync is available from the Press admin panel via a "Sync Marketplace Registry" button.

### Version resolution

At install time, Press resolves which branch to use:

1. Get the site's current Frappe version (e.g. `15.18.2`) from the site's Release Group.
2. Query `Marketplace App Registry Version` for the app. Evaluate each `frappe_range` using `packaging.specifiers.SpecifierSet(frappe_range).contains(site_frappe_version)`. Take the first match.
3. Find or create an `AppSource` record for `(source_url, resolved_branch)`. Proceed through the existing `ReleaseGroup → DeployCandidate → Deploy` flow. Press never does a raw git clone — the Agent process on the target server does that as part of the deploy pipeline.
4. Recursively resolve and install any `requires` dependencies declared in the matched entry using the same lookup.

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
| `[[app.versions]]` entries | `Marketplace App Registry Version` child table (new doctype; one row per entry; fields: `frappe_range`, `branch`, `requires`) |

> Fields marked ⚠️ need to be added to the `Marketplace App` doctype. The current doctype tracks publisher via the `team` Link field — `publisher` and `publisher_name` are new flat fields for the registry-native flow.

### Conflict resolution

If `app.toml` is valid and passes schema check, the Press record is overwritten unconditionally. If an `apps/<id>/` directory is removed from the registry (merged removal PR), the corresponding `Marketplace App` is set to `status = Disabled`. It is not deleted — subscription and billing history must be preserved.

## Telemetry Emission

Press knows exactly when apps change state on customer sites because it manages all deployments. PostHog calls are added in `press/press/doctype/site/site.py` at the points where app installs and uninstalls are confirmed — the same places where `marketplace_app_hook` is already called (lines ~844, ~847). The `marketplace_app_hook` itself runs publisher-defined install/uninstall scripts; telemetry is separate and must not go inside that hook.

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
