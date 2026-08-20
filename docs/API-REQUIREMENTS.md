# Platform Health Report — API Requirements

**Status:** Draft for review  
**Date:** 20 August 2026  
**Audience:** Integration / platform owners, AMS, D&T Global Integration Team  
**Live report:** https://bks002.github.io/aramex-platform-health-report/

This document lists the APIs required to replace Google Sheet CSV feeds and embedded sample data with live, automated sources. Reviewers should confirm:

1. Endpoint coverage is complete for Home, platforms, Tickets, and Regional tasks.
2. Payload fields match operational reporting (units, success rules, CPI discarded).
3. Which upstream systems will own each feed.
4. What remains manual until a source system exists.

---

## 1. Purpose

The report is an **Integration Control / platform health** view for four platforms:

| Platform ID | Display name | Volume unit in UI |
|-------------|----------------|-------------------|
| `infolink` | Infolink | Millions (M) |
| `boomi` | Boomi | Thousands (K) |
| `shipping` | Shipping API | Millions (M) |
| `cpi` | CPI | Thousands (K) |

Plus:

- ServiceNow ticket desk
- Regional tasks (Freight & Logistics, GCC, Express)

APIs must return **canonical JSON**. The UI formats units (M / K, seconds) and should not re-parse CSV.

---

## 2. Current vs target

| Area | Today | Target |
|------|--------|--------|
| Weekly / monthly metrics | Week Over Week + Monthly Message Processed sheets | Metrics APIs |
| Current snapshot, highlights, risks, integrations | Summary of Insights sheet | Executive / current API |
| Day-wise volume | Data tab + Daily Performance Dashboard | Daily metrics API |
| Live hourly | Hourly Data sheet | Hourly metrics API (faster poll) |
| KPIs | KPI sheet | KPI API |
| Tickets | Tickets sheet | Ticket summary API |
| Regional tasks | Embedded dummy CSV | Regional tasks API |

---

## 3. Shared conventions

Apply to **all** APIs.

| Rule | Requirement |
|------|-------------|
| Protocol | HTTPS REST, JSON (`application/json`) |
| Time | ISO-8601; timestamps with offset (e.g. `+04:00`) |
| Dates | `YYYY-MM-DD` |
| Message volumes | Absolute integers (`messageCount`). Do **not** send millions in the payload. UI converts to M/K. |
| Latency | Milliseconds (`latencyMs`) |
| Percentages | Numeric, 0–100, two decimal places where used (e.g. `97.85`) |
| Platform IDs | `infolink` \| `boomi` \| `shipping` \| `cpi` |
| Health | `green` \| `yellow` \| `red` \| `gray` |
| Auth | Backend holds secrets. GitHub Pages must not store API keys. |
| CORS | Allow the Pages origin: `https://bks002.github.io` |
| Cache | `Cache-Control` appropriate to cadence (see §7) |
| Errors | JSON `{ "error": { "code": "...", "message": "..." } }` with 4xx/5xx |

### Health (recommended server-side, same as report today)

| Condition | Health |
|-----------|--------|
| Sev1 > 0 **or** success < 95% **or** failure > 5% | `red` |
| Sev2 > 0 **or** success < 97% **or** failure > 3% | `yellow` |
| Otherwise (metrics present) | `green` |
| No metrics | `gray` |

UI copy: healthy failure ≤ 3%, success ≥ 97%, no Sev1/Sev2.

### Platform-specific calculation rules

**Infolink reported success (day / hourly)**

```
reportedSuccessCount = pureSuccessCount + partialFailureCount + duplicateCount
```

KPI “Integration Success Rate” uses this framing (partial / duplicate treated as success with errors).

**CPI processed total**

Discarded (timer-overlap drops) is **not** part of processed volume:

```
messageCount = successCount + failureCount
```

Do not include `discardedCount` in `messageCount`. Optional field `discardedCount` may still be returned for ops, but the report does not show it in day-wise totals/tables.

---

## 4. API inventory

Base path (proposed): `/api/v1`

| # | Method | Path | Replaces | Poll |
|---|--------|------|----------|------|
| 1 | GET | `/platforms/current` | Summary of Insights (snapshot + narrative) | 5 min |
| 2 | GET | `/platforms/{id}/daily` | Data tab + Daily Performance Dashboard | 5 min |
| 3 | GET | `/platforms/{id}/weekly` | Week Over Week Summary | 5 min |
| 4 | GET | `/platforms/{id}/monthly` | Monthly Message Processed + WoW rollup | 5 min |
| 5 | GET | `/platforms/hourly` | Hourly Data | 30–60 s |
| 6 | GET | `/kpis` | KPI sheet | 5 min |
| 7 | GET | `/tickets/summary` | Tickets sheet | 1–5 min |
| 8 | GET | `/regional-tasks` | Embedded dummy CSV | 5 min |
| 9 | GET | `/report` *(optional)* | Bundle of 1–4, 6–8 for page load | 5 min |

Hourly stays a **separate** call. Do not fold it into `/report` if the home page polls it every minute.

---

## 5. Endpoint specifications and sample payloads

### 5.1 Current platform summary

`GET /api/v1/platforms/current`

Used by: Home cards, command bar period, integrations, highlights/risks, leadership attention.

```json
{
  "reportingPeriod": {
    "id": "2026-W31",
    "label": "28th July – 3rd August",
    "startDate": "2026-07-28",
    "endDate": "2026-08-03",
    "isLatest": true
  },
  "platforms": [
    {
      "id": "infolink",
      "name": "Infolink",
      "health": "yellow",
      "messageCount": 15680000,
      "successPercentage": 97.85,
      "failurePercentage": 2.14,
      "latencyMs": 7100,
      "availabilityPercentage": 99.99,
      "integrationsRunning": 24815,
      "integrationsLabel": "Integrations",
      "severity": { "sev1": 0, "sev2": 0, "sev3": 36 },
      "highlights": "Stable volume; latency watch.",
      "risks": "Success below 99% target when partials are excluded from pure success."
    },
    {
      "id": "boomi",
      "name": "Boomi",
      "health": "green",
      "messageCount": 859176,
      "successPercentage": 99.91,
      "failurePercentage": 0.09,
      "latencyMs": 40,
      "availabilityPercentage": 99.9,
      "integrationsRunning": 148,
      "integrationsLabel": "Integrations",
      "severity": { "sev1": 0, "sev2": 0, "sev3": 4 },
      "highlights": "Processing remained stable.",
      "risks": null
    },
    {
      "id": "shipping",
      "name": "Shipping API",
      "health": "green",
      "messageCount": 73020043,
      "successPercentage": 100,
      "failurePercentage": 0,
      "latencyMs": 172,
      "availabilityPercentage": 99.9,
      "apiProcessed": 73020043,
      "integrationsLabel": "API Processed",
      "severity": { "sev1": 0, "sev2": 0, "sev3": 0 },
      "highlights": "No failures in the reporting week.",
      "risks": null
    },
    {
      "id": "cpi",
      "name": "CPI",
      "health": "green",
      "messageCount": 11277,
      "successPercentage": 99.25,
      "failurePercentage": 0.75,
      "latencyMs": 1200,
      "availabilityPercentage": 99.9,
      "integrationsRunning": 40,
      "integrationsLabel": "Integrations",
      "severity": { "sev1": 0, "sev2": 0, "sev3": 1 },
      "highlights": null,
      "risks": "Discarded volume remains high vs processed messages."
    }
  ],
  "leadershipAttention": null,
  "generatedAt": "2026-08-03T09:00:00+04:00"
}
```

**Review notes**

- `integrationsRunning` / `apiProcessed` are **current reporting week only** (no history in WoW today).
- Highlights, risks, and leadership attention have no machine source today — confirm owner (manual CMS vs derived).

---

### 5.2 Day-wise performance

`GET /api/v1/platforms/{platformId}/daily?from=2026-07-28&to=2026-08-03`

Used by: platform **current week** day-wise charts and table.

```json
{
  "platformId": "cpi",
  "period": { "from": "2026-07-28", "to": "2026-08-03" },
  "days": [
    {
      "date": "2026-07-27",
      "messageCount": 1489,
      "successCount": 1481,
      "failureCount": 8,
      "discardedCount": 765,
      "successPercentage": 99.46,
      "failurePercentage": 0.54
    }
  ],
  "weekTotals": {
    "messageCount": 11277,
    "successCount": 11192,
    "failureCount": 85,
    "discardedCount": 5359
  },
  "rules": {
    "discardedIncludedInMessageCount": false
  },
  "generatedAt": "2026-08-03T09:00:00+04:00"
}
```

**Infolink day row (extra fields)**

```json
{
  "date": "2026-07-27",
  "messageCount": 2596536,
  "internalCount": 377716,
  "externalCount": 2218820,
  "pureSuccessCount": 2200399,
  "partialFailureCount": 197874,
  "duplicateCount": 153585,
  "failureCount": 44678,
  "successCount": 2551858,
  "successPercentage": 98.28,
  "failurePercentage": 1.72
}
```

`successCount` = pure + partial + duplicate.

**Review notes**

- Day-wise UI is shown only for the **latest reporting week**.
- Boomi / Shipping: success and failure counts only (no discarded / partial columns).

---

### 5.3 Weekly history

`GET /api/v1/platforms/{platformId}/weekly?limit=12`

Used by: 4-week trend cards, historical week snapshot, vs previous week.

```json
{
  "platformId": "infolink",
  "weeks": [
    {
      "id": "2026-W30",
      "label": "21st – 27th July",
      "startDate": "2026-07-21",
      "endDate": "2026-07-27",
      "messageCount": 15680000,
      "successPercentage": 97.85,
      "failurePercentage": 2.14,
      "latencyMs": 7100,
      "availabilityPercentage": 99.99,
      "severity": { "sev1": 0, "sev2": 0, "sev3": 36 }
    },
    {
      "id": "2026-W29",
      "label": "14th – 20th July",
      "startDate": "2026-07-14",
      "endDate": "2026-07-20",
      "messageCount": 16200000,
      "successPercentage": 97.89,
      "failurePercentage": 2.11,
      "latencyMs": 7300,
      "availabilityPercentage": 97.6,
      "severity": { "sev1": 1, "sev2": 2, "sev3": 40 }
    }
  ],
  "generatedAt": "2026-08-03T09:00:00+04:00"
}
```

Order: oldest → newest (or document newest-first; UI can reverse).

---

### 5.4 Monthly history

`GET /api/v1/platforms/{platformId}/monthly?limit=12`

Used by: Month mode snapshot and vs previous month.

**Server aggregation (must match report)**

| Metric | Aggregation |
|--------|-------------|
| `messageCount`, Sev1/2/3 | **Sum** of weeks in the month |
| Success %, failure %, latency, availability | **Average** of weeks that have values |

Monthly Message Processed sheet can override messages / success / failure when it is the official month total.

```json
{
  "platformId": "boomi",
  "months": [
    {
      "id": "2026-07",
      "label": "July 2026",
      "messageCount": 3290000,
      "successPercentage": 99.92,
      "failurePercentage": 0.08,
      "averageLatencyMs": 40,
      "averageAvailabilityPercentage": 99.92,
      "severity": { "sev1": 0, "sev2": 0, "sev3": 14 },
      "weeksIncluded": 4
    }
  ],
  "generatedAt": "2026-08-03T09:00:00+04:00"
}
```

---

### 5.5 Live hourly

`GET /api/v1/platforms/hourly?date=2026-08-20`

Used by: Home “Live hourly performance”.

```json
{
  "date": "2026-08-20",
  "asOfHour": 11,
  "platforms": [
    {
      "platformId": "boomi",
      "title": "Boomi",
      "hours": [
        {
          "hour": 10,
          "timestamp": "2026-08-20T10:00:00+04:00",
          "messageCount": 24115,
          "successCount": 24092,
          "failureCount": 23,
          "averageLatencyMs": 42
        }
      ]
    }
  ],
  "generatedAt": "2026-08-20T11:05:00+04:00"
}
```

**Infolink hourly:** `successCount` includes partial failure (same as day-wise).

**Review notes**

- Return hours **up to the current local hour** only.
- Missing platform sections → UI shows “Awaiting data”.

---

### 5.6 KPIs

`GET /api/v1/kpis?platformId=infolink`

Used by: platform KPI cards (current week only for Infolink / Boomi / CPI).

```json
{
  "platformId": "infolink",
  "kpis": [
    {
      "id": "integration-success-rate",
      "name": "Integration Success Rate",
      "operator": "gte",
      "target": 99,
      "current": 97.85,
      "unit": "percentage",
      "assessment": "watch",
      "margin": -1.15,
      "statusDescription": "Success 97.85% (including 7.46% success with errors). Failure 2.14%. Duplicate 6.20%.",
      "actionPlan": "Investigate subscribers with processing time over 10 minutes.",
      "actionDate": "2026-08-03"
    },
    {
      "id": "message-latency",
      "name": "Message Latency",
      "operator": "lte",
      "target": 30000,
      "current": 7100,
      "unit": "milliseconds",
      "assessment": "met",
      "margin": 22900,
      "statusDescription": "7.1 sec",
      "actionPlan": null,
      "actionDate": null
    },
    {
      "id": "platform-availability",
      "name": "Platform Availability",
      "operator": "gte",
      "target": 99.9,
      "current": 99.9,
      "unit": "percentage",
      "assessment": "met",
      "margin": 0,
      "statusDescription": "99.90%",
      "actionPlan": null,
      "actionDate": null
    }
  ],
  "generatedAt": "2026-08-03T09:00:00+04:00"
}
```

`operator`: `gte` | `lte`  
`assessment`: `met` | `watch` | `miss` | `info`

Shipping also has **API Error Rate** (`target` < 1%).

---

### 5.7 Ticket summary

`GET /api/v1/tickets/summary`

Used by: command bar ticket count, Tickets tab.

```json
{
  "totals": {
    "total": 42,
    "inProgress": 12,
    "closed": 28,
    "unassigned": 2
  },
  "previousPeriod": {
    "total": 47,
    "inProgress": 15,
    "closed": 30,
    "unassigned": 2
  },
  "members": [
    {
      "name": "Naveen",
      "total": 8,
      "inProgress": 3,
      "closed": 5
    }
  ],
  "generatedAt": "2026-08-20T11:00:00+04:00"
}
```

---

### 5.8 Regional tasks

`GET /api/v1/regional-tasks`

Used by: Regional tasks tab. Replaces sample dummy data.

Query: `?region=freight|gcc|express` optional.

```json
{
  "regions": [
    {
      "id": "freight",
      "name": "Freight & Logistics Region",
      "tasks": [
        {
          "id": "FL-001",
          "serial": 1,
          "owner": "Naveen",
          "title": "IRAQ EDI Integration - Azadea Boomi Process",
          "status": "in_progress",
          "startedDate": "2026-07-20",
          "endDate": null,
          "updatedDate": "2026-07-28",
          "remarks": "Updated in Test env; UAT with Azadea scheduled this week"
        }
      ]
    }
  ],
  "summary": {
    "total": 15,
    "inProgress": 9,
    "completed": 3,
    "blocked": 3
  },
  "generatedAt": "2026-08-20T11:00:00+04:00"
}
```

`status`: `in_progress` | `completed` | `blocked` | `on_hold`

---

### 5.9 Optional report bundle

`GET /api/v1/report?period=2026-W31`

```json
{
  "current": {},
  "weeklyByPlatform": {},
  "monthlyByPlatform": {},
  "dailyByPlatform": {},
  "kpisByPlatform": {},
  "tickets": {},
  "regionalTasks": {},
  "generatedAt": "2026-08-20T11:00:00+04:00"
}
```

Omit `hourly` from this payload.

---

## 6. Upstream systems (to confirm)

| Feed | Likely source | Owner to confirm |
|------|----------------|------------------|
| Infolink volumes / latency / availability | Infolink / internal telemetry | |
| Boomi | Boomi AtomSphere Platform API | |
| CPI | SAP CPI Message Processing Logs / OData | |
| Shipping API | API gateway / APM / custom metrics | |
| Tickets | ServiceNow Table API | |
| Regional tasks | Jira / ADO / ServiceNow / tracker | |
| Highlights, risks, leadership, KPI action plans | Manual until a CMS exists | AMS |

A **BFF (backend for frontend)** should sit in front of vendor APIs: aggregate, apply Infolink/CPI rules, compute health and KPI assessment, cache.

---

## 7. Cadence

| API | Suggested refresh |
|-----|-------------------|
| `/platforms/hourly` | 30–60 seconds while Home is open |
| `/tickets/summary` | 1–5 minutes |
| All other report APIs | 5 minutes (aligned with current dashboard) |

---

## 8. Implementation sequence (recommended)

1. Weekly metrics + current snapshot (Home + WoW cards).
2. Daily (current-week platform pages).
3. Hourly (Home live charts).
4. Monthly (month mode).
5. KPIs + tickets.
6. Regional tasks (replace dummy data).

Keep Google Sheet publish as **fallback** until each API is signed off.

---

## 9. Review checklist

Please mark Accept / Change / Defer.

| # | Decision | Accept | Change | Defer |
|---|----------|--------|--------|-------|
| R1 | Eight core endpoints (+ optional `/report`) are sufficient | | | |
| R2 | Volumes as absolute counts; UI formats M/K | | | |
| R3 | Latency in milliseconds | | | |
| R4 | Infolink success = pure + partial + duplicate | | | |
| R5 | CPI discarded excluded from processed total | | | |
| R6 | Health rules in §3 | | | |
| R7 | Integrations / narrative remain current-week-only | | | |
| R8 | Upstream owners in §6 | | | |
| R9 | Auth model (public Pages + BFF vs private report) | | | |
| R10 | Regional task source system | | | |

**Comments / requested changes**

_Add review comments here._

---

## 10. Out of scope (this draft)

- OpenAPI 3.1 file (follow-up after this review)
- Auth / Entra ID design
- Vendor-specific field mapping worksheets
- Changing GitHub Pages hosting model

---

*Prepared for review by Annual Maintenance Support · ALL D&T – Global – Integration Team.*
