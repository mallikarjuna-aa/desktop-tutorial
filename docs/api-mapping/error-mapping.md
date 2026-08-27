# Error Mapping Strategy

This mapping normalizes source/target API errors to a common integration contract.

## Common Error Contract

```json
{
  "errorCode": "STRING_CODE",
  "httpStatus": 400,
  "message": "Human-readable summary",
  "retryable": false,
  "sourceSystem": "source-api|target-api",
  "details": {}
}
```

## Source/Target to Common Mapping

| Source | Target | Common `errorCode` | HTTP Status | Retryable | Handling Strategy |
|---|---|---|---:|---|---|
| `VALIDATION_ERROR` | `INVALID_REQUEST` | `ERR_VALIDATION` | 400 | No | Return field-level validation details to caller. |
| `UNAUTHORIZED` | `AUTH_FAILED` | `ERR_AUTH` | 401 | No | Stop flow; require credential/token refresh. |
| `FORBIDDEN` | `ACCESS_DENIED` | `ERR_FORBIDDEN` | 403 | No | Stop flow; raise access-policy incident. |
| `NOT_FOUND` | `RESOURCE_NOT_FOUND` | `ERR_NOT_FOUND` | 404 | No | Log and skip record for bulk; fail sync request. |
| `CONFLICT` | `DUPLICATE_RESOURCE` | `ERR_CONFLICT` | 409 | No | Apply idempotency check and retry with merge rule if configured. |
| `TOO_MANY_REQUESTS` | `RATE_LIMIT_EXCEEDED` | `ERR_RATE_LIMIT` | 429 | Yes | Retry with exponential backoff + jitter. |
| `TIMEOUT` | `GATEWAY_TIMEOUT` | `ERR_TIMEOUT` | 504 | Yes | Retry up to max attempts; move to dead-letter after threshold. |
| `SERVER_ERROR` | `INTERNAL_ERROR` | `ERR_UPSTREAM` | 500 | Yes | Retry transiently; create incident if persistent. |
| `SERVICE_UNAVAILABLE` | `SERVICE_UNAVAILABLE` | `ERR_UNAVAILABLE` | 503 | Yes | Retry with circuit-breaker policy. |

## Retry Policy
- Max attempts: **3**
- Backoff: `2s`, `4s`, `8s` + random jitter
- Non-retryable: 4xx (except 429)
