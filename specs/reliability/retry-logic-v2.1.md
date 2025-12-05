# Retry Logic Specification V2.1

## Overview

Defines retry logic with exponential backoff and jitter for all API calls.

## Jitter Formula

```yaml
backoff_type: exponential_with_jitter
jitter_formula: "delay = base * 2^retry * (0.5 + 0.5 * random())"
base_delay_ms: 1000
```

## Per-API Configurations

| API | Max Attempts | Timeout (ms) | Notes |
|-----|--------------|--------------|-------|
| Perplexity | 4 | 180000 | Deep Research is slow |
| Claude | 3 | 60000 | Standard config |
| ChatPRD | 2 | 30000 | Fast API |
| Devin | 2 | 300000 | Builds are slow |
| GitHub | 3 | 10000 | Critical dependency |

## Retry Conditions

The following HTTP status codes trigger automatic retry with exponential backoff:

| Status Code | Description | Retry Behavior |
|-------------|-------------|----------------|
| 429 | Too Many Requests | Retry with backoff, respect Retry-After header if present |
| 500 | Internal Server Error | Retry with backoff |
| 502 | Bad Gateway | Retry with backoff |
| 503 | Service Unavailable | Retry with backoff, respect Retry-After header if present |
| 504 | Gateway Timeout | Retry with backoff |

## No-Retry Conditions

The following HTTP status codes should NOT trigger retry (fail immediately):

| Status Code | Description | Reason |
|-------------|-------------|--------|
| 400 | Bad Request | Client error - request is malformed |
| 401 | Unauthorized | Authentication required - retry won't help |
| 403 | Forbidden | Permission denied - retry won't help |
| 404 | Not Found | Resource doesn't exist - retry won't help |
| 422 | Unprocessable Entity | Validation error - fix request first |

## Implementation Notes

1. **Jitter Purpose**: Prevents thundering herd problem when multiple clients retry simultaneously
2. **Retry-After Header**: When present in 429/503 responses, use the specified delay instead of calculated backoff
3. **Circuit Breaker**: Consider implementing circuit breaker pattern for repeated failures (>50% failure rate over 10 requests)
4. **Logging**: Log all retry attempts with attempt number, delay, and error details for debugging
