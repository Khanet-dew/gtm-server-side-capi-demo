# gtm-server-side-capi-demo
It's just the gtm server-side tracking demo.
# GTM Server-Side Tracking + Meta CAPI Demo

A complete implementation of **GTM Server-Side Tracking** with **Meta Conversions API (CAPI)** — built as a learning portfolio and reusable template for real client work.

## What This Project Does

Replaces browser-side pixel tracking with a server-side architecture that:

- **Bypasses ad blockers** — events are sent server-to-server, not browser-to-vendor
- **Survives Safari ITP** — first-party cookies set by your own server, not restricted by browser
- **Improves Meta CAPI data quality** — purchase events reach Meta even when browser pixel is blocked
- **Reduces data loss** — typically recovers 20–40% of events lost to browser-side blocking

## Architecture

```
User's Browser
    │
    │  dataLayer.push({ event: 'purchase', ... })
    ▼
GTM Web Container (GTM-XXXXXXX)
    │
    │  HTTP POST → transport_url (your server, not Google/Meta directly)
    ▼
GTM Server Container (GCP Cloud Run)
    │  YOUR_SERVER_URL
    │
    ├──► GA4 Client → parses incoming request
    │
    └──► Meta CAPI Tag → POST to graph.facebook.com/events
                         server-to-server, no browser involved
```

## Repo Structure

```
gtm-server-side-capi-demo/
├── README.md                          # This file
├── gtm-config/
│   ├── web-container.json             # GTM Web Container export (import-ready)
│   └── server-container.json         # GTM Server Container export (import-ready)
├── test-page/
│   └── gtm-test-page.html            # Local test page with simulated e-commerce events
├── tracking-spec/
│   └── tracking-spec-template.xlsx   # Tracking spec template to send developers
└── docs/
    └── setup-guide.md                # Step-by-step setup instructions
```

## Quick Start

### Prerequisites

- Google Tag Manager account
- GCP (Google Cloud Platform) account — free tier works for dev/test
- Meta Business account with a Pixel

### Step 1 — Replace placeholders

Search and replace all placeholders in the files before use:

| Placeholder | Replace with |
|---|---|
| `YOUR_GTM_ACCOUNT_ID` | Your GTM Account ID (found in GTM URL) |
| `YOUR_GTM_WEB_CONTAINER_ID` | Your Web Container ID e.g. `GTM-XXXXXXX` |
| `YOUR_GTM_SERVER_CONTAINER_ID` | Your Server Container ID e.g. `GTM-XXXXXXX` |
| `YOUR_SERVER_URL` | Your GCP Cloud Run URL e.g. `https://xxxxx-uc.a.run.app` |
| `YOUR_GA4_MEASUREMENT_ID` | Your GA4 Measurement ID e.g. `G-XXXXXXXXXX` |
| `YOUR_META_PIXEL_ID` | Your Meta Pixel ID e.g. `1234567890123456` |
| `YOUR_META_ACCESS_TOKEN` | Your Meta CAPI Access Token (never commit the real value) |

### Step 2 — Deploy Server Container on GCP

1. Go to [tagmanager.google.com](https://tagmanager.google.com)
2. Create new container → Target platform: **Server**
3. Choose **Automatically provision tagging server**
4. Select or create a GCP project
5. Wait ~3–5 minutes → get your `YOUR_SERVER_URL`

### Step 3 — Import GTM containers

**Web Container:**
1. GTM → Admin → Import Container
2. Select `gtm-config/web-container.json`
3. Choose workspace → Import

**Server Container:**
1. GTM → Admin → Import Container
2. Select `gtm-config/server-container.json`
3. Choose workspace → Import

### Step 4 — Test locally

1. Install [VS Code](https://code.visualstudio.com) + Live Server extension
2. Open `test-page/gtm-test-page.html` → right-click → **Open with Live Server**
3. Enable GTM Preview Mode on both Web and Server containers
4. Click any button on the test page
5. Verify in Server Preview: **Meta CAPI - Purchase → Fired**
6. Verify in Meta Events Manager → Test Events: **Purchase → Processed**

## Key Concepts

### Why Transport URL?

Normally GTM Web Container sends events directly to Google/Meta servers. Ad blockers intercept these requests because they recognize vendor domains.

By setting `transport_url` to your own server URL, the browser sees a first-party request to your domain — ad blockers don't block it.

### Why Meta CAPI deduplication matters

Both browser pixel AND server CAPI send the same event. Without deduplication, Meta counts it twice. The `event_id` field must be identical between both — this project handles it automatically via the `event_id` parameter in `gtm-test-page.html`.

### Client vs Tag (Server Container)

| Component | Role |
|---|---|
| **Client** (GA4 Client) | Receives incoming HTTP request, parses it into event data |
| **Tag** (Meta CAPI Tag) | Reads event data, forwards to Meta via server API |

## Files Included

### `gtm-config/web-container.json`
Contains:
- **GA4 - Configuration Tag** — sets `transport_url` and `server_container_url` pointing to your server
- **GA4 - Event - Purchase Tag** — fires `purchase` event to server
- **Custom Event - Purchase Trigger** — fires on `dataLayer.push({ event: 'purchase' })`

### `gtm-config/server-container.json`
Contains:
- **GA4 Client** — claims and parses incoming GA4 requests
- **Meta CAPI Tag** — forwards purchase events to Meta Conversions API

### `test-page/gtm-test-page.html`
A simulated e-commerce shop with:
- 4 demo products with real dataLayer push on every button click
- Real-time event log showing dataLayer state
- Events: `view_item`, `add_to_cart`, `purchase`, `begin_checkout`, `sign_up`, and more
- `event_id` auto-generated on every purchase for Meta deduplication

### `tracking-spec/tracking-spec-template.xlsx`
Production-ready tracking spec with 5 events pre-documented:
- `purchase` (P0) — full e-commerce parameters + Meta dedup fields
- `add_to_cart` (P0)
- `view_item` (P1)
- `begin_checkout` (P1)
- `sign_up` (P1)

Each event includes: trigger conditions, dataLayer parameters, code examples, and acceptance criteria for developer handoff.

## Tech Stack

| Layer | Technology |
|---|---|
| Tag Management | Google Tag Manager (Web + Server Container) |
| Server Infrastructure | GCP Cloud Run (auto-provisioned) |
| Tracking Destination | Meta Conversions API, GA4 Measurement Protocol |
| Test Environment | HTML + JavaScript, VS Code Live Server |

## Deployment Options

GTM Server Container runs as a Docker image — deployable anywhere:

| Platform | Difficulty | Notes |
|---|---|---|
| GCP Cloud Run (auto) | Easiest | One-click from GTM UI |
| GCP Cloud Run (manual) | Easy | More config control |
| AWS App Runner | Medium | Good if client is on AWS |
| Docker on any VPS | Medium | Full self-hosted control |

All options use the same `CONTAINER_CONFIG` environment variable from GTM Server Container settings.

## Acknowledgements

- Meta CAPI Tag template by [stape.io](https://stape.io)
- GTM Server-Side documentation by [Simo Ahava](https://simoahava.com)

---

Built by [Khanet-dew (Devver)](https://github.com/Khanet-dew) as a learning portfolio project.
