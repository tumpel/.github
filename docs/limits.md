# Limits

Every endpoint and every host has limits. The control plane refuses traffic past a limit before it writes anything, so a runaway host or client stops at the limit and cannot run up storage on your Cloudflare account.

The defaults fit a small host that runs about nine boxes at once. A tripped limit fails the operation and marks the host unhealthy. Treat it as a signal to review the host or raise the limit. Never hide it with retries.

## The limits

| Limit                        | Counted per | Default             | Set by                | Refusal                   |
| ---------------------------- | ----------- | ------------------- | --------------------- | ------------------------- |
| Request body                 | request     | 4 MB                | fixed                 | 413 `BodyTooLarge`        |
| Tenant config plan and apply | request     | 16 MB               | fixed                 | 413 `BodyTooLarge`        |
| GitHub webhook body          | request     | 25 MB               | fixed                 | 413 `BodyTooLarge`        |
| Requests                     | tenant      | 600 a minute        | `readsPerMinute`      | 429 `RateLimited`         |
| Requests that write          | tenant      | 120 a minute        | `writesPerMinute`     | 429 `RateLimited`         |
| Signed webhook deliveries    | tenant      | 600 a minute        | fixed                 | 429 `RateLimited`         |
| Host channel frames          | host        | 600 a minute        | `hostFramesPerMinute` | host marked `RateLimited` |
| Rows written                 | tenant      | 3,300,000 an hour   | `configured`          | 429 `UsageLimitReached`   |
| Rows read                    | tenant      | 100,000,000 an hour | `configured`          | 429 `UsageLimitReached`   |

Every request counts against the request rate, and a request that changes state also counts against the write rate. A 429 carries `retry-after`, and its detail names the limit it hit.

The fields in the Set by column live in `TUMPEL_LIMITS`, the limits profile of the control plane Worker. It is one JSON object in the control plane's `wrangler.jsonc`. A profile that does not match its schema fails every request with an error that names the field.

## When a host passes its frame limit

1. The control plane marks the host `RateLimited` and emits a `HostHealthChanged` event.
2. It fails every running turn on that host with the reason `RateLimited`.
3. It offers that host no new work until a full minute passes without a refused frame.
4. After three trips within an hour, the host goes `Unhealthy`. It stays there until the supervisor restarts or someone restores it with `POST /v1/hosts/{hostId}/restore-health`.

The supervisor backs off with jitter after each refusal. It exits after 10 failed backoffs.

## Size the limits to your host

Raise the limits when your host runs more boxes than the default. A host with 20 boxes at once sends about 2.2 times the frames and requests of a nine-box host. Multiply each limit by your box count over nine, then add headroom. Size the per-tenant limits for all of the tenant's hosts together.

Each per-minute rate appears twice in `wrangler.jsonc`: once in `TUMPEL_LIMITS` and once as the `limit` of its rate limiting binding. The two must match. Each stage's `env` block repeats both, so change the block of every stage you run.

| Field                 | Binding             |
| --------------------- | ------------------- |
| `readsPerMinute`      | `TENANT_READ_RATE`  |
| `writesPerMinute`     | `TENANT_WRITE_RATE` |
| `hostFramesPerMinute` | `HOST_FRAME_RATE`   |

This profile sizes a tenant with one 20-box host, at 2.5 times the defaults:

```jsonc
"TUMPEL_LIMITS": {
  "defaultTier": "Configured",
  "freeTenantsPerAccount": 1,
  "configured": [
    { "measure": "RowsWritten", "window": "Hour", "limit": 8250000 },
    { "measure": "RowsRead", "window": "Hour", "limit": 250000000 }
  ],
  "readsPerMinute": 1500,
  "writesPerMinute": 300,
  "hostFramesPerMinute": 1500
}
```

```jsonc
"ratelimits": [
  { "name": "TENANT_READ_RATE", "namespace_id": "1001", "simple": { "limit": 1500, "period": 60 } },
  { "name": "TENANT_WRITE_RATE", "namespace_id": "1002", "simple": { "limit": 300, "period": 60 } },
  { "name": "HOST_FRAME_RATE", "namespace_id": "1003", "simple": { "limit": 1500, "period": 60 } },
  { "name": "WEBHOOK_RATE", "namespace_id": "1004", "simple": { "limit": 600, "period": 60 } }
]
```

Keep each binding's `namespace_id` as your stage has it. Deploy the control plane to apply the change. Higher limits buy capacity, and they let a runaway host spend more of your Cloudflare bill before the control plane stops it.

## Stay on the free plan

A tenant runs on one of two tiers. `Configured` gets the `configured` limits above. `Free` keeps the tenant inside the allowance that the Cloudflare free plan includes each day, and refuses traffic past it.

| Free limit   | Default with one free tenant |
| ------------ | ---------------------------- |
| Requests     | 40,000 a day, 400 a minute   |
| Rows written | 80,000 a day                 |
| Rows read    | 4,000,000 a day              |

Each free tenant gets 80% of the daily allowance, divided by `freeTenantsPerAccount`. Set that field to the number of free tenants on your Cloudflare account. Only the tenant's own Durable Object storage counts towards these limits.

`defaultTier` sets the tier of a new tenant. An existing tenant keeps its tier in the `tier` column of the `tenants` table in the stage's D1 database:

```sh
npx wrangler d1 execute <database> --remote \
  --command "UPDATE tenants SET tier = 'Free' WHERE tenant_id = '<tenantId>'"
```

The tenant picks up its new tier the next time its Durable Object starts.

## Watch usage and host health

- `GET /v1/metrics` and the MCP tool `tumpel_metrics_catalog` return `usage`: the tenant's tier and, for each usage limit, `used`, `limit` and `resetsAt` in the current window.
- The tenant event log gets `UsageNearLimit` at 80% of a usage limit and `UsageLimitReached` at the limit, each once per window. The MCP tool `tumpel_events_wait` reads them.
- Each host carries `health` (`Healthy`, `RateLimited` or `Unhealthy`) and `reportedHealth`, the health its supervisor reports in its heartbeat. `GET /v1/hosts/{hostId}` returns both, and every change emits `HostHealthChanged`.
- `tumpel host list-usage` shows each host's box slots, running and warm boxes, load and free memory. Compare it with the box count your limits assume.
