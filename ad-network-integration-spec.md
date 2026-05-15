# Technical Integration Specification for Ad Networks

This document describes the API your ad network needs to expose for integration with our rotation engine.

---

## General Requirements

| Parameter | Value |
|---|---|
| Protocol | HTTPS |
| Data format | JSON |
| Encoding | UTF-8 |
| Authentication | Any method that works for you. Examples: `apiKey` as a query parameter, `Authorization: Bearer <token>` header, or a custom header (`X-Api-Key`, `X-Auth-Token`). |
| Response format | The "Zone ad code" endpoint (see 1.2) accepts one of two shapes: (1) both `html` and `url` fields at once, one may be empty; (2) one field plus an explicit `type` (`html` or `url`). All other endpoints return regular JSON. |

---

## Part 1. Zones and ad code

> **Note on endpoints.** All URLs below (`/api/v1/sites`, `/api/v1/zones/{zone_id}/code`, `/api/v1/stats`) are **structural examples**, not required paths. Use whatever paths suit your stack — just send us the actual URLs and preserve the semantics (what each method returns and what parameters it accepts).

### 1.1. List of sites and ad zones

**Endpoint**
```
GET /api/v1/sites
```

**Description**
Returns all of the partner's sites along with their ad zones.

**Example response**
```json
{
  "sites": [
    {
      "site_id": "123",
      "domain": "example.com",
      "zones": [
        {
          "zone_id": 1,
          "name": "Header 728x90"
        },
        {
          "zone_id": 2,
          "name": "Sidebar 300x250"
        }
      ]
    },
    {
      "site_id": "456",
      "domain": "another-site.com",
      "zones": [
        {
          "zone_id": 3,
          "name": "Footer 970x250"
        },
        {
          "zone_id": 4,
          "name": "In-Content 300x600"
        }
      ]
    }
  ]
}
```

> The response structure can differ — what matters is the semantics: site → array of zones with `zone_id` and a name.

---

### 1.2. Zone ad code

**Endpoint**
```
GET /api/v1/zones/{zone_id}/code
```

**Description**
Returns the ad creative for the zone — an HTML snippet, a URL for embedding, or both.

> The dual-format response below applies **only to this endpoint**. All other API methods return regular JSON.

Two response shapes are acceptable:

**Option 1 — both fields at once (`html` + `url`), one may be empty / `null`**
```json
{
  "zone_id": 1,
  "html": "<script src=\"https://ads.example.com/ad.js\"></script>",
  "url": ""
}
```
```json
{
  "zone_id": 2,
  "html": null,
  "url": "https://ads.example.com/render?zone_id=2"
}
```

**Option 2 — one field plus an explicit `type`**
```json
{
  "zone_id": 1,
  "type": "html",
  "html": "<script src=\"https://ads.example.com/ad.js\"></script>"
}
```
```json
{
  "zone_id": 2,
  "type": "url",
  "url": "https://ads.example.com/render?zone_id=2"
}
```

---

## Part 2. Statistics

Statistics are reported per ad zone, sliced by:

- **dates** (`date`)
- **countries** (`country`)
- **sub-ids** (`sub_id`)

Multi-dimensional grouping in a single request is preferred but not required. If you don't support it, return one dimension per request and we'll make multiple calls.

### 2.1. Fetching statistics

**Endpoint**
```
GET /api/v1/stats
```

**Query parameters**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `date_from` | `YYYY-MM-DD` | yes | Start of the period |
| `date_to` | `YYYY-MM-DD` | yes | End of the period |
| `group_by` | comma-separated list | yes | Grouping fields: `date`, `country`, `sub_id` |
| `zone_id` | int | no | Filter by a specific zone |

**Example requests**

```
GET /api/v1/stats?date_from=2025-01-01&date_to=2025-01-07&group_by=date,country
GET /api/v1/stats?date_from=2025-01-01&date_to=2025-01-07&group_by=date,sub_id
GET /api/v1/stats?date_from=2025-01-01&date_to=2025-01-07&group_by=date,country,sub_id&zone_id=1
```

**Example response**
```json
{
  "data": [
    {
      "date": "2025-01-01",
      "country": "US",
      "sub_id": "abc123",
      "zone_id": 1,
      "impressions": 12000,
      "clicks": 340,
      "revenue": 45.67
    },
    {
      "date": "2025-01-01",
      "country": "DE",
      "sub_id": "def456",
      "zone_id": 1,
      "impressions": 5400,
      "clicks": 120,
      "revenue": 18.10
    }
  ]
}
```

> Field names and response shape above are **just an example**. Yours may use different names (`shows`/`views` instead of `impressions`, `earnings`/`payout` instead of `revenue`, nested objects instead of a flat list, etc.). What matters is that the response contains the grouping fields and the core metrics: impressions, clicks, revenue. Send us your actual schema.

---

## Part 3. Passing `sub_id` and `keywords`

We critically depend on passing **subID** (for source tracking) and **keywords** (for targeting and context) along with traffic. Please describe how your format accepts these parameters — examples below.

### 3.1. Via a URL zone (`type: "url"`)

We expect either macro substitution or query parameters:

```
https://ads.example.com/render?zone_id=2&sub_id={SUB_ID}&keywords={KEYWORDS}
```

Example of a substituted URL:
```
https://ads.example.com/render?zone_id=2&sub_id=user_42_landing_A&keywords=casino,slots,bonus
```

### 3.2. Via an HTML zone (`type: "html"`)

We expect macros inside the snippet, which we replace before serving to the user:

```html
<script
  src="https://ads.example.com/ad.js"
  data-zone="zone_1"
  data-sub-id="{SUB_ID}"
  data-keywords="{KEYWORDS}">
</script>
```

Or as script parameters:

```html
<script src="https://ads.example.com/ad.js?zone=zone_1&sub_id={SUB_ID}&keywords={KEYWORDS}"></script>
```

### 3.3. What we need from you

To validate the integration, please give us access to a **live or test account** with real data so we can exercise all the endpoints:

1. **Auth credentials** (API key / token / login+password — depending on your chosen method).
2. **At least one site** in the account with ad zones configured — so that `/sites` and `/zones/{id}/code` return non-empty responses.
3. **Accumulated traffic** on those zones (impressions, clicks, revenue) — so we can hit `/stats` for a past period and see real numbers grouped by date, country, and `sub_id`.

Without live data we can't validate responses or grouping formats.

---

## Part 4. API limits

To configure rotation and stats collection on our side, please send us the following information about your API:

1. **Rate limit for `/stats`** — the maximum number of requests per unit of time (per second / minute / hour). The limit only matters for stats — we call `/sites` and `/zones/{id}/code` very rarely.
2. **Response code on overage** — which HTTP status is returned (`429 Too Many Requests`, `503`, something else) and whether there's a header indicating retry timing (`Retry-After`, `X-RateLimit-Reset`, etc.).
3. **Maximum stats period per request** — how many days can be requested in a single `/stats` call (for example, up to 31 days, up to 90 days). If there's a row-count limit on the response, share that too.
4. **Behavior beyond the period limit** — error response or automatic truncation of the result.
