# Rate Limiter Tier Policies

## Overview

The Rate Limiter Tier Policies capability provides multi-tier, policy-driven request throttling for the Revora Backend. Designed specifically for critical onboarding and authentication surfaces such as the Startup Registration endpoint (`POST /api/v1/startup/register`), this system enables differentiated throughput quotas based on caller trust profiles while maintaining fail-safe security boundaries.

---

## Tier Definitions & Quotas

The system establishes three explicit rate-limiting tiers configured in `src/middleware/startupAuthRateTierPolicy.ts`:

| Tier | Request Limit | Window Duration | Description | Target Actors |
| :--- | :---: | :---: | :--- | :--- |
| **`standard`** | **5 requests** | 15 minutes (900,000 ms) | Default tier for anonymous or unverified public requests. Provides strong anti-abuse and brute-force protection. | General public, unauthenticated clients |
| **`trusted`** | **10 requests** | 15 minutes (900,000 ms) | Elevated quota for verified partner applications and integrations. | Partner portals, verified integrations |
| **`internal`** | **25 requests** | 15 minutes (900,000 ms) | High-throughput tier for internal Revora microservices, batch provisioning, and administrative jobs. | Automated deployment runners, internal services |

---

## Request & Response Header Specification

### Request Headers

Clients specify their desired rate-limiting tier and authentication secret using custom Revora headers:

| Header | Required | Format | Description |
| :--- | :---: | :--- | :--- |
| `x-revora-rate-tier` | No | String (`standard`, `trusted`, `internal`) | Specifies the desired rate tier. Defaults to `standard` if omitted or unparseable. Case-insensitive and trimmed. |
| `x-revora-tier-secret` | Conditional | String | Shared authentication secret required to claim elevated tiers (`trusted` or `internal`). |

### Response Headers

On both successful responses (e.g. `201 Created`) and rate-limited responses (`429 Too Many Requests`), the middleware sets standard informational headers:

| Header | Example | Description |
| :--- | :--- | :--- |
| `X-RateLimit-Limit` | `10` | The total number of requests permitted in the current time window for the resolved tier. |
| `X-RateLimit-Remaining` | `4` | The number of remaining requests allowed in the current window. Decrements deterministically. |
| `X-RateLimit-Reset` | `1790772000` | Unix epoch timestamp (in seconds) indicating when the current window resets. |
| `X-RateLimit-Tier` | `trusted` | The actual tier resolved and applied by the server after cryptographic/secret evaluation. |
| `Retry-After` | `842` | *(Only present on 429)* Seconds remaining until the rate-limit window resets and new requests are accepted. |

---

## Security Model & Assumptions

The rate limiter implementation adheres to defense-in-depth principles:

### 1. Fail-Safe Silent Downgrade
- When a caller requests an elevated tier (`trusted` or `internal`) without providing the secret or providing an invalid secret, the system **silently downgrades** the request to the `standard` tier.
- It does **not** return a 401 or 403 error. This prevents timing or enumeration attacks that could reveal whether an elevated tier exists or whether a secret is close to matching.
- The `X-RateLimit-Tier` response header reflects `standard`, transparently confirming the enforced policy.

### 2. Isolated Bucket Namespaces
- Each tier operates under its own isolated key prefix:
  - `startup-auth:standard:<ip>`
  - `startup-auth:trusted:<ip>`
  - `startup-auth:internal:<ip>`
- If an attacker attempts to spoof elevated tiers with incorrect secrets, the requests consume only the `standard` bucket quota. The legitimate `trusted` or `internal` buckets remain completely unaffected.

### 3. Server-Side Secret Verification
- Elevated tiers require the `STARTUP_AUTH_TIER_SECRET` environment variable to be configured on the server.
- If the server has no `STARTUP_AUTH_TIER_SECRET` set, all elevated requests fail safe to the `standard` tier.
- Secret comparisons strip accidental leading and trailing whitespace to prevent misconfiguration.

### 4. Pre-Validation Enforcement
- Rate-limiting middleware is mounted **before** request body parsing and business logic validation.
- Invalid requests (e.g. missing required fields resulting in `400 Bad Request`) consume rate-limiting quota tokens. This prevents resource exhaustion attacks where invalid requests could otherwise be flooded without penalty.

### 5. Cross-Route Isolation
- Rate limits applied to `/api/v1/startup/register` are completely isolated from other platform endpoints.
- Critical operational routes (such as `/health`, `/ready`, and `/live`) are unaffected by rate limiting on authentication routes.

---

## Architecture & Implementation

### Core Components

1. **`createStartupAuthTierLimiter`** (`src/middleware/startupAuthRateTierPolicy.ts`):
   - Resolves caller tier from request headers and environment variables.
   - Instantiates isolated `createRateLimitMiddleware` handlers for each tier.
   - Exposes `resolveTier(req)` and `reset()` helpers for deterministic testing and administration.

2. **`createRateLimitMiddleware`** (`src/middleware/rateLimit.ts`):
   - Fixed-window rate limiter supporting both IP-based and user-based keying.
   - Backed by `InMemoryRateLimitStore` by default, with an extensible `RateLimitStore` interface suitable for distributed backends (e.g. Redis).

3. **Application Wiring** (`src/index.ts`):
   - The tier limiter is integrated into `apiRouter.post("/startup/register", ...)` in `createApp()`.
   - Injected via `AppDependencies.startupTierLimiter` to allow mocking or custom storage in external harnesses.

```typescript
// Injection pattern in src/index.ts
export interface AppDependencies {
  healthQuery?: typeof dbQuery;
  healthStatus?: typeof dbHealth;
  startupTierLimiter?: ReturnType<typeof createStartupAuthTierLimiter>;
}

export function createApp(dependencies: AppDependencies = {}): express.Express {
  // ...
  const startupTierLimiter =
    dependencies.startupTierLimiter ?? createStartupAuthTierLimiter();

  apiRouter.post(
    "/startup/register",
    startupTierLimiter.middleware,
    createStartupRegisterHandler(),
  );
  // ...
}
```

---

## Testing & Verification

The suite provides deterministic, comprehensive test coverage exceeding 95%:

- **Integration Tests** (`src/routes/health.test.ts`):
  - Standard tier defaults when headers are absent.
  - Quota exhaustion tests (blocking the 6th standard request, 11th trusted request, and 26th internal request).
  - Silent downgrade verification when secrets are incorrect, empty, or whitespace.
  - Counter isolation between spoofed standard requests and legitimate trusted requests.
  - Pre-validation abuse protection (400 responses consuming quota before 429).
  - Independent health endpoint resilience during active registration throttling.
  - Dependency injection and store reset capabilities.

- **Unit Tests** (`src/middleware/startupAuthRateTierPolicy.test.ts`):
  - Exhaustive unit tests for `resolveTier` covering all permutation of valid, invalid, empty, and omitted headers.
  - Response header structure and error message validation.

- **Store & Core Tests** (`src/middleware/rateLimit.test.ts`):
  - Fixed-window reset timing, remaining count calculation, and header generation.

---

## Production Deployment Considerations

- **Multi-Instance Environments**: The default `InMemoryRateLimitStore` is process-local. In multi-pod or multi-region deployments, implement `RateLimitStore` backed by Redis using `INCR` and `EXPIREAT` to share counters across replicas.
- **Proxy Configuration**: Revora Backend runs behind trusted reverse proxies (e.g. Cloudflare, AWS ALB). Ensure `app.set('trust proxy', 1)` is enabled so client IP resolution correctly parses `X-Forwarded-For`.
