# API — Subscription Tiers & Webhooks

Developer reference for the tier system and webhook infrastructure.

## Subscription Tiers

| | Free | Starter | Pro | Enterprise |
|---|---|---|---|---|
| Organizations owned | 1 | 3 | 10 | Custom |
| API keys / org | 1 | 5 | 20 | Custom |
| Webhooks / org | 2 | 5 | 20 | Custom |
| Requests / day | 1,000 | 50,000 | 500,000 | Custom |
| Requests / sec | 10 | 50 | 200 | Custom |

- **Enterprise overrides** -- reads custom limits from the `tier_config` JSON column on the organization row. Unset keys default to unlimited (`-1`).
- **Quota enforcement** -- returns HTTP 429 with error code `QUOTA_EXCEEDED`.
- **Rate limit headers** -- `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` (included on every API-key-authenticated response).

### Key source files

- `api/internal/core/domain/organization.go` -- tier definitions, `GetTierLimits()`
- `api/internal/gateway/middleware/quota_adapter.go` -- quota enforcement middleware
- `api/internal/core/domain/api_usage.go` -- usage tracking model

## Webhooks

### Event types

| Event | Trigger |
|---|---|
| `game.started` | Game status transitions to `live` |
| `game.ended` | Game status transitions to `final` |
| `game.score_updated` | Home or away score changes |
| `game.event_created` | New game event recorded (play, foul, etc.) |
| `stats.updated` | Player or team statistics recalculated |

### Delivery payload

```json
{
  "event_type": "game.started",
  "data": { "game_id": "<uuid>", "sport": "basketball" },
  "timestamp": "2024-03-04T12:00:00.000000000Z",
  "delivery_id": "<uuid>"
}
```

Headers sent with each delivery:

- `Content-Type: application/json`
- `X-GoStats-Signature: sha256=<hmac_hex>` -- HMAC-SHA256 of the raw body using the webhook secret
- `X-GoStats-Event: <event_type>`
- `X-GoStats-Delivery: <delivery_id>`

### Retry policy

- **Backoff schedule** -- 1 min, 5 min, 30 min, 2 h, 24 h (exponential)
- **Max attempts** -- 5
- **Auto-disable** -- webhook is disabled after 10 consecutive failures (`FailureCount >= 10`)

### Architecture

- Sport workers publish events to Redis Streams via `WebhookJobPublisher`
- Webhook worker consumes from the stream, fetches the secret from the DB (not stored in Redis), signs and delivers
- SSRF protection via `safedialer.go` (blocks private IPs)
- URLs must use HTTPS
- Dead letter queue for exhausted retries

### Management endpoints

| Method | Path | Notes |
|---|---|---|
| `POST` | `/api/v1/organizations/{orgId}/webhooks` | Create (secret returned only at creation) |
| `GET` | `.../webhooks` | List |
| `GET` | `.../webhooks/{id}` | Get |
| `PATCH` | `.../webhooks/{id}` | Update |
| `DELETE` | `.../webhooks/{id}` | Delete |
| `POST` | `.../webhooks/{id}/test` | Send test delivery |
| `GET` | `.../webhooks/{id}/deliveries` | Recent delivery history |

### Key source files

- `api/internal/core/domain/webhook.go` -- domain model, event types, retry logic
- `api/internal/webhooks/deliverer.go` -- HTTP delivery, HMAC signing, payload format
- `api/internal/webhooks/safedialer.go` -- SSRF-safe transport
- `api/internal/gateway/handlers/webhook_management.go` -- CRUD endpoints
- `api/cmd/worker-webhook/main.go` -- webhook worker service
- `api/internal/adapters/redis/webhook_streams.go` -- Redis stream pub/sub
