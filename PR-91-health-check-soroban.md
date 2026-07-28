# PR: Extend /health to Check Soroban RPC Reachability

## Summary

Previously, `/health` only reported process liveness (uptime, version, network name), giving a false "ok" signal even when the Soroban RPC endpoint was completely unreachable. Since `SorobanService` is now a real external dependency used by `/api/v1/chain/*`, the health endpoint must reflect Soroban's availability to be useful for load balancers, orchestrators, and monitoring.

## Changes

### Modified Files

- **`src/health/health.controller.ts`**
  - Injects `SorobanService` (exported from `SorobanModule`)
  - The `check()` method is now `async`
  - Calls `sorobanService.getHealth()` with a **5-second timeout** via a `withTimeout` helper
  - On success (Soroban RPC returns `{ status: "healthy" }`):
    - Returns HTTP 200 with `status: "ok"` and the Soroban health payload in a `soroban` field
  - On failure (timeout, network error, or non-healthy status):
    - Returns **HTTP 503** with `status: "degraded"` and `soroban.status: "unreachable"`
  - Uses `timer.unref()` on the timeout so it does not block process exit

- **`src/health/health.module.ts`**
  - Imports `SorobanModule` to make `SorobanService` available for injection into `HealthController`

- **`test/health.e2e-spec.ts`**
  - Updated to accept both healthy (200) and degraded (503) responses
  - Validates that `soroban` field is present in both cases with the correct status value
  - Increased `beforeAll` timeout to 30s to accommodate Soroban RPC call

## Response Shapes

### Healthy (`200 OK`)

```json
{
  "status": "ok",
  "service": "vortex-backend",
  "version": "0.1.0",
  "network": "stellar-testnet",
  "uptime": 1234.56,
  "soroban": { "status": "healthy" }
}
```

### Degraded (`503 Service Unavailable`)

```json
{
  "status": "degraded",
  "service": "vortex-backend",
  "version": "0.1.0",
  "network": "stellar-testnet",
  "uptime": 1234.56,
  "soroban": { "status": "unreachable" }
}
```

## Design Decisions

### Why 5-second timeout?
Soroban RPC calls can occasionally be slow under load (rate limiting, network congestion). 5 seconds provides enough time for a legitimate response while keeping the health check responsive. The timeout uses `timer.unref()` so it does not prevent the process from shutting down.

### Why 503 instead of 200 with degraded field?
NestJS's `ServiceUnavailableException` maps to HTTP 503, which is the correct status code for a service that is alive but cannot serve requests due to a dependency failure. Load balancers and orchestrators (Kubernetes, etc.) understand 503 and will stop routing traffic to the instance.

### Why not make the entire health check fail fast?
The health endpoint should still report basic process info (uptime, version, network) even when Soroban is down. Returning a full payload with a `status` field allows operators to see exactly what is degraded.

## Testing

```
npm run lint       ✓
npm run typecheck  ✓
npm test           ✓ 18 passed (3 suites)
npm run test:e2e   ✓ 23 passed (5 suites, including updated health.e2e-spec.ts)
```

The e2e test verifies both the successful and degraded paths depending on whether a real Soroban RPC endpoint is reachable from the test environment.

Closes #91
