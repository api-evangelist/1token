---
name: 1Token 1ndex strategy overview
description: >-
  Read the aggregate 1ndex institutional crypto strategy overview — platform counts, per-strategy
  performance summaries and historical time series — from 1Token's one public, anonymous API,
  without mangling its nanosecond timestamps.
api: openapi/1token-1ndex-openapi.yml
operations:
  - getPublicStrategyOverview
generated: '2026-09-05'
method: generated
source: openapi/1token-1ndex-openapi.yml + https://1token.tech/api/1ndex/v1/README.md
---

# 1Token 1ndex strategy overview

## What this API is, and is not

`getPublicStrategyOverview` is the **only** operation 1Token publishes publicly. It returns
**aggregates** about the 1ndex strategy-discovery platform. The provider states the boundary
explicitly: this contract does **not** expose customer-specific data, write operations, 1Token CAM
product interfaces, or third-party APIs 1Token consumes. If a task needs any of those, this API is
the wrong surface — stop and say so rather than probing for undocumented paths.

## Authentication

None. Do not send an Authorization header, an API key, or a cookie.

## Call it

```
GET https://1ndex.1token.tech/api/v1/public/strategy-overview
Accept: application/json
```

All three parameters are optional:

| Parameter | Type | Notes |
| --- | --- | --- |
| `start_time` | integer int64 | Start of the history window, Unix timestamp in **nanoseconds** |
| `end_time` | integer int64 | End of the history window, Unix timestamp in **nanoseconds** |
| `strategy_type` | string | e.g. `DeltaNeutral`. **Not** a fixed enum — discover valid values from `strategy_types[].type` in an unfiltered response |

Bounded example:

```bash
curl --get 'https://1ndex.1token.tech/api/v1/public/strategy-overview' \
  --data-urlencode 'start_time=1783900800000000000' \
  --data-urlencode 'end_time=1783987200000000000' \
  --data-urlencode 'strategy_type=DeltaNeutral' \
  --header 'Accept: application/json'
```

## Read the response

- `platform_statistics` — `total_investors`, `total_strategies`, `total_trading_teams`.
- `strategy_types[]` — one entry per category: `type`, `description`, `active_teams`,
  `active_strategies`, `summary_data` (performance plus `sharpe_ratio` and `max_drawdown`), and
  `historical_time_series_data[]`.
- Performance fields (`accum_nav`, `accum_pnl`, `24h_pnl`, `7d_pnl`, `30d_pnl`, `mtd_pnl`,
  `qtd_pnl`, `ytd_pnl`, and the `*_ann_pnl` annualized variants) are **raw numbers**. The contract
  deliberately defines no rounding, percentage or sign convention. Do not assume a scale, do not
  render a `%` sign you cannot justify, and do not convert without saying what you assumed.

## The one trap: nanosecond timestamps

`time` and `update_time` are 19-digit Unix **nanosecond** integers. They exceed
`Number.MAX_SAFE_INTEGER`, so any JavaScript or JSON parser using IEEE-754 doubles will silently
corrupt them. Read the sibling RFC 3339 strings instead — `time_str` and `update_time_str` — whenever
exact time matters. When you must send a timestamp, multiply seconds by 1e9 and send an integer.

## Errors

Not RFC 9457. A `400` returns `application/json` with a stable two-field envelope:

```json
{"code": "invalid_parameter", "message": "Invalid parameter: invalid start_time: expected nanosecond Unix timestamp"}
```

`invalid_parameter` is the only code named anywhere in the contract. Nothing else is enumerated —
no 404, no 429, no 5xx — so treat any other non-200 as an undocumented transport failure and back
off; do not retry tightly.

## Rate limits and courtesy

There is no published quota and no rate-limit header, and 1Token states the endpoint is best-effort
with **no availability SLA**. The provider's own instruction is: keep request frequency low, cache
results, and use `start_time`/`end_time`/`strategy_type` rather than re-downloading the full
document. Follow it — a polite caller is the only rate limiting that exists here.

## Idempotency and reversibility

Read-only. There is nothing to make idempotent and nothing to reverse. Ignore the
`Idempotency-Key` entry in the CORS preflight — it is gateway configuration, not a contract.
