# Telemetry & Analytics

## Overview

App developers get a GitHub-style analytics dashboard showing adoption of their app across Frappe Cloud. Press emits PostHog events whenever marketplace apps are installed, updated, or removed on customer sites. Developers query this data through a dashboard in Frappe Cloud.

PostHog is already integrated in Press via `press/utils/telemetry.py`. Marketplace telemetry extends the existing integration with no new infrastructure.

## Architecture

```
Frappe Cloud Site
      │
      │  install / uninstall / update
      ▼
  Press (manages all site deployments)
      │
      │  emit PostHog events
      ▼
  PostHog
      │
      │  query via PostHog API (filtered by publisher)
      ▼
  Developer Dashboard
  /marketplace/publisher/analytics
```

## Events

### `marketplace_app_installed`

Fired when a site installs a marketplace app for the first time.

The `version` field is the app's semver string (e.g. `"15.0.1"`). It comes from the app's `__version__` variable, defined in the app package's `__init__.py` and exposed via `hooks.py` as `app_version = __version__`. Press reads this from the cloned App Release at build time. It is set by the app developer and is separate from the git commit hash the registry tracks.

```json
{
  "event": "marketplace_app_installed",
  "distinct_id": "<site-name>",
  "properties": {
    "app": "hrms",
    "publisher": "frappe",
    "version": "15.0.1",
    "frappe_version": "15",          // major version integer as string, from Frappe Version doctype
    "region": "ap-south-1",
    "subscription_type": "Free",
    "$ip": null
  }
}
```

### `marketplace_app_uninstalled`

Fired when a site removes a marketplace app.

```json
{
  "event": "marketplace_app_uninstalled",
  "distinct_id": "<site-name>",
  "properties": {
    "app": "hrms",
    "publisher": "frappe",
    "version": "15.0.1",
    "frappe_version": "15",
    "region": "ap-south-1",
    "$ip": null
  }
}
```

### `marketplace_app_updated`

Fired when a site upgrades the app to a newer version.

```json
{
  "event": "marketplace_app_updated",
  "distinct_id": "<site-name>",
  "properties": {
    "app": "hrms",
    "publisher": "frappe",
    "from_version": "15.0.0",
    "to_version": "15.0.1",
    "frappe_version": "15",
    "region": "ap-south-1",
    "$ip": null
  }
}
```

## Developer Dashboard Metrics

Available at `/marketplace/publisher/analytics` in Frappe Cloud. All queries are filtered server-side by `publisher` — developers only see their own apps.

### Total Active Installs

Sites with the app installed and not subsequently uninstalled.

**Query:** Count `distinct_id`s with `marketplace_app_installed` and no later `marketplace_app_uninstalled`, grouped by `app`.

### Installs Over Time

New installs per week over the last 12 months. Displayed as a line chart.

**Query:** Weekly breakdown of `marketplace_app_installed` events filtered by `app`.

### Version Distribution

Percentage of active sites on each version. Displayed as a bar chart.

**Query:** Group active sites by `version` property on their most recent event.

### Geographic Distribution

Active installs by region. AWS region mapped to a display name.

**Query:** Group active sites by `region` property.

| AWS Region | Display Name |
|------------|-------------|
| `ap-south-1` | India |
| `us-east-1` | US East |
| `us-west-2` | US West |
| `eu-west-1` | Europe |
| `ap-southeast-1` | Southeast Asia |
| `me-south-1` | Middle East |

### Uninstall Rate

Percentage of all-time installs that uninstalled. Signal for app quality and onboarding issues.

**Query:** `total(marketplace_app_uninstalled) / total(marketplace_app_installed)` for the app.

## Privacy

- `distinct_id` is the Frappe Cloud site name, not a user email or personal identifier
- No site content or customer data is included in any event
- `region` is the AWS region of the Frappe Cloud server, not the customer's IP or location
- `$ip` is explicitly `null` on all events — PostHog does not geolocate by IP

## Access Control

The developer dashboard applies a mandatory `publisher` filter server-side in Press. Developers cannot query other publishers' data. The Frappe team has access to all publishers' data for marketplace health monitoring.

## PostHog Integration

The existing `capture()` function in `press/utils/telemetry.py` currently has the signature:

```python
def capture(event, app, distinct_id=None):
```

It does not accept a `properties` dict. For marketplace telemetry events, this function needs to be **extended** to accept arbitrary properties:

```python
def capture(event, app, distinct_id=None, properties=None):
```

Example call after the extension:

```python
capture(
    event="marketplace_app_installed",
    app="fc_marketplace",
    distinct_id=site.name,
    properties={
        "app": app_name,
        "publisher": publisher,
        "version": version,         # from app's __version__ in __init__.py
        "frappe_version": frappe_version,
        "region": site.cluster.region,
        "subscription_type": subscription_type,
        "$ip": None,
    }
)
```

No new PostHog project is needed. Existing Press PostHog project is used, with `publisher` and `app` properties for filtering.
