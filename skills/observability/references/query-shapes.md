# Funnel, retention and ad-hoc queries: shapes and pitfalls

Read when building a funnel, retention or ad-hoc query.

## Time ranges

`timeRange` is `{ "type": "relative", "value": "last_7d" }` or `{ "type": "absolute", "from": "<ISO 8601>", "to": "<ISO 8601>" }`. Relative values: `last_15m`, `last_1h`, `last_24h`, `last_7d`, `last_30d`, `last_90d`, `last_180d`, `last_365d`, `today`, `yesterday`, `this_week`, `this_month`, `this_quarter`, `this_year`. Anything else is refused.

## Funnel

`swiss-knife:funnel_report` arguments:

```json
{
  "app_project_id": "<uuid>",
  "steps": [
    { "eventName": "signup_completed" },
    { "eventName": "purchase_completed" }
  ],
  "conversion_window": 86400,
  "timeRange": { "type": "relative", "value": "last_30d" }
}
```

- `conversion_window` is in seconds, at most 30 days (2592000).
- Steps are ordered: a user counts at step N only after steps 1 to N-1 within the window.
- Optional `breakdown` is `{ "source": "event", "name": "properties.plan" }`; take the property name from `swiss-knife:property_list`.
- A step filter is `{ "property": { "source": "event", "name": "properties.plan" }, "operator": "eq", "values": ["pro"] }`.

## Retention

`swiss-knife:retention_report`: `cohort_event` is the event that puts a user in a cohort (first occurrence); `return_event` defaults to any event; `granularity` is `day`, `week` or `month`; `periods` up to 24.

## Ad-hoc query

`swiss-knife:report_run` takes `measures` (each with `id`, optional `eventName`, `func` such as `count`, `uniq`, `sum`, `avg`; `sum`, `avg`, `min`, `max` and the percentiles need a `property`), `timeRange`, optional `filters`, `breakdown` (`property`, `limit` up to 50), `formula` (`expression` over measure ids, for example `B/A*100`), `granularity` (`minute` to `month`) and `limit`.

## Reading the answer

- Funnel `steps[]` carry `count`, `conversionRate` (from the previous step) and `overallRate` (from step 1).
- State the window, the step names and the counts, not only the rates.
- A short window or a new app gives small counts; say so instead of drawing a trend.
