# REST API security and stability review

## Snapshot of current behavior

### 1) API auth is a single static key
- Every `/api/*` request is authorized in `ApiController::__construct()` by reading a `token` value from `$_GET` or `$_POST` and passing it to `ApiModel::validateKey()`. If the key is active and matches, the request is allowed. There is no per-user context, no scopes, and no endpoint-level authorization differentiation today.

### 2) Token in query string is currently supported by design
- Because the token is read from `$_GET`, clients can send requests like `/api/plants/remove?token=...&plant=...`.
- This pattern is functional, but high risk for operational leaks (URL logging, browser history, reverse proxy logs, shared screenshots, etc.).

### 3) Mutating API endpoints accept `ANY` method
- In `app/config/routes.php`, most API endpoints are configured with `ANY`, including destructive endpoints such as `/api/plants/remove`, `/api/tasks/remove`, `/api/inventory/remove`, and more.
- Combined with query token auth, this makes accidental or unsafe invocations easier than needed.

### 4) API key model is coarse-grained
- API keys are generated as random hashes and can be toggled active/inactive.
- There is no expiration, no purpose/scope, no per-key metadata (last used, created by, label), and no rotation policy support.

## Risks this creates

1. **Token exfiltration risk via URLs**
   - Query-string tokens are commonly captured in access logs and browser telemetry.
2. **Excessive blast radius**
   - A single leaked token can execute all API actions, including delete operations.
3. **Weak method discipline**
   - `ANY` allows sensitive actions via GET-like invocation patterns.
4. **Operational blind spots**
   - No fine-grained audit trail per key/action/IP means harder incident response.

## Deployment context and threat model (current usage)

- Access is currently remote through Tailscale to a locally hosted container (phone client connects over tailnet).
- Data sensitivity is relatively low (plant/workspace data), so the goal is **practical safety and stability**, not maximum-hardening enterprise controls.
- There is a functional automation requirement: OpenClaw needs API access for heartbeat/reminder features (watering reminders etc.).

### Practical implication
- The most important near-term improvement is to remove token exposure from URLs while keeping API access simple and reliable for trusted devices/services.
- Security changes should avoid breaking existing automation clients.

## Stability/security hardening options (recommended order)

## Phase 1 (quick wins, low migration cost)
1. **Require token in `Authorization` header**
   - Support `Authorization: Bearer <token>`.
   - Keep query token support temporarily behind a feature flag + deprecation warnings.
2. **Constrain HTTP methods per route**
   - `GET` for reads only.
   - `POST/PATCH/DELETE` for mutating operations.
3. **Add server-side request logging for API actions**
   - Log: timestamp, endpoint, key id (not full token), status code, source IP, user-agent.

## Phase 2 (defense in depth)
1. **Introduce scoped API keys**
   - Example scopes: `plants.read`, `plants.write`, `inventory.read`, `inventory.write`, `backup.export`.
2. **Add key lifecycle controls**
   - Expiration timestamps, rotation reminders, last-used tracking, optional one-time reveal.
3. **Rate limiting and lockout**
   - Per-key and per-IP request budget + anomaly throttling.

## Phase 3 (strongest posture)
1. **Signed request scheme (HMAC) or short-lived tokens**
   - Reduce replay risk and limit value of leaked credentials.
2. **Optional allowlist controls**
   - Network/IP restrictions for automation keys.
3. **Versioned API contract (`/api/v2`)**
   - Introduce stronger auth semantics without breaking existing automation.

## Practical migration path

1. Add header-token support first while preserving old behavior.
2. Emit warning response metadata when `token` is provided via query string.
3. Update native/mobile client and OpenClaw heartbeat client to header auth.
4. Keep access limited to trusted network paths (e.g., Tailscale tailnet) and disable query token auth after clients are migrated.
5. Narrow route methods and add scope checks for write/delete endpoints.

## Recommended “good enough” baseline for this setup

1. Header token only (`Authorization: Bearer`) for all API clients.
2. Keep Tailscale as the exposure boundary (no public internet API exposure).
3. Restrict mutating routes to non-GET methods.
4. Add lightweight request logging (endpoint + key id + status + source IP).
5. Use a dedicated OpenClaw key and rotate if leaked/suspected.

## Suggested implementation anchors in this codebase

- **Auth entry point:** `app/controller/api.php` constructor.
- **Key validation:** `app/models/ApiModel.php`.
- **Method restrictions:** `app/config/routes.php`.
- **Admin key management UX:** `app/controller/admin.php` and `app/views/admin.php`.
