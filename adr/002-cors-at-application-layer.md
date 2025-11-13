# ADR-002: Configure CORS at Application Layer Using @fastify/cors

## Status

Accepted (Supersedes ADR-001)

## Context

ADR-001 attempted to configure CORS at the Traefik ingress layer using Headers Middleware, following perceived industry best practices. After implementation and testing, this approach failed in our environment.

### What Went Wrong with ADR-001

**Traefik Middleware Did Not Work As Expected:**

1. **OPTIONS Requests Not Intercepted**
   - Traefik Headers Middleware did NOT intercept OPTIONS preflight requests
   - OPTIONS requests returned 404 from Traefik instead of 204 with CORS headers
   - Backend needed to handle OPTIONS, but Traefik stripped CORS headers from responses

2. **Backend CORS Headers Stripped**
   - Direct pod test: `OPTIONS /health` → 204 with CORS headers ✅
   - Through Traefik: `OPTIONS /health` → 404 ❌
   - Through Traefik: `GET /health` → NO CORS headers ❌
   - Traefik was removing backend CORS headers from responses

3. **Catch-22 Situation**
   - Can't use backend CORS: Traefik strips headers
   - Can't use pure ingress CORS: Traefik doesn't intercept OPTIONS
   - Would need complex debugging of Traefik configuration

### ADR-001 Incorrect Claims

**Line 35 Claim:** "Fastify documentation warns against wildcards and recommends ingress-layer configuration for production"

**Reality:** Fastify documentation ([fastify-cors](https://github.com/fastify/fastify-cors)) only warns against wildcard origins with credentials. It does NOT recommend ingress-layer vs application-layer configuration. This was a misinterpretation.

### Industry Best Practice Clarification

The actual best practice is: **Choose ONE layer (application OR ingress), not both.**

**NOT:** "Always use ingress layer"

Evidence from our research:

- EdgeX Foundry: _"Implementers should choose one or the other [gateway or microservice], not both"_
- The key is avoiding duplicate headers, not which layer you choose
- Many production systems successfully use application-layer CORS

## Decision

We will **configure CORS at the application layer using @fastify/cors** and **remove CORS from Traefik middleware**.

### Implementation Details

**Application Configuration:**

- Use `@fastify/cors` v11.1.0 with async registration
- Configure via `CORS_ORIGINS` environment variable
- Support credentials, standard methods, and headers

**Infrastructure Configuration:**

- Remove `cors-headers` middleware from IngressRoutes
- Remove Traefik Headers Middleware resources (no longer needed)
- Configure `CORS_ORIGINS` in deployment patches per environment

### Rationale

1. **Works Reliably**
   - @fastify/cors is proven and battle-tested
   - Direct pod testing confirmed it works correctly
   - No Traefik configuration issues to debug

2. **Simpler Architecture**
   - CORS configuration lives with application code
   - No dependency on ingress controller behavior
   - Easier to test and validate

3. **Better Service Autonomy**
   - Microservice owns its security policy
   - Can be tested independently
   - No infrastructure coupling

4. **Race Condition Fix**
   - PR #160 fixed async plugin registration
   - `await app.register(cors, ...)` prevents race conditions
   - Headers now consistently applied

5. **Proven Approach**
   - Many production Fastify applications use this pattern
   - Well-documented and supported
   - No special ingress controller requirements

## Consequences

### Positive

- ✅ CORS works reliably without Traefik configuration issues
- ✅ Service owns its security policy (microservices principle)
- ✅ Easier to test (direct pod testing validates CORS)
- ✅ No ingress controller coupling
- ✅ Simpler deployment (no middleware resources)
- ✅ Configuration lives with application code
- ✅ Works with any ingress controller (Traefik, NGINX, etc.)

### Negative

- ⚠️ Preflight OPTIONS requests hit backend (not handled at edge)
- ⚠️ Each service must configure CORS independently
- ⚠️ CORS origins configured via environment variables (not centralized)

### Neutral

- CORS configuration is environment-specific via Kustomize
- Backend handles all CORS logic
- Standard Fastify pattern

## Implementation

### Backend Changes (Already Complete)

PR #160 in platform-backend:

- Added async `createApp()` function
- Configured `@fastify/cors` with proper async registration
- Added `CORS_ORIGINS` config from environment variable
- Fixed race condition that caused intermittent CORS failures

### Infrastructure Changes (In Progress)

PR #40 in steady-manifests:

- Remove `cors-headers` middleware reference from platform-backend IngressRoutes
- Add `CORS_ORIGINS` environment variable to platform-backend deployment patches
- **Keep middleware files for Centrifugo's continued use**

Environment-specific origins for platform-backend:

- Stage: `https://app-stage.steady.ops.last-try.org,https://miniapp-stage.steady.ops.last-try.org`
- Prod: `https://app.steady.ops.last-try.org,https://miniapp.steady.ops.last-try.org`

**Note on Centrifugo**: Centrifugo (WebSocket server at `ws-stage.steady.ops.last-try.org` and `ws.steady.ops.last-try.org`) continues using Traefik CORS middleware. WebSocket connections have different CORS semantics than HTTP. This configuration will be evaluated separately in a future ADR.

## Testing

**Verification Steps:**

1. Test OPTIONS preflight: `curl -X OPTIONS [api-url] -H "Origin: [allowed-origin]"`
   - Should return 204 with CORS headers
2. Test actual request: `curl -X GET [api-url] -H "Origin: [allowed-origin]"`
   - Should return 200 with CORS headers
3. Test unauthorized origin: Should NOT include CORS headers
4. Test credentials: Verify `Access-Control-Allow-Credentials: true`

## Security Considerations

### Best Practices Applied

1. **No Wildcards with Credentials**
   - Explicit origin list only
   - Configured via environment variable per environment

2. **HTTPS-Only Origins**
   - All configured origins use HTTPS
   - No HTTP in production

3. **Standard Methods and Headers**
   - Only methods actually used: GET, POST, PUT, DELETE, PATCH, OPTIONS
   - Minimal header list: Content-Type, Authorization, X-Requested-With, X-CSRF-Token

4. **Reasonable Cache**
   - 24-hour maxAge for preflight caching
   - Balances performance and flexibility

## Alternatives Considered

### Alternative 1: Debug Traefik Configuration

**Pros:**

- Might achieve ingress-layer CORS
- Centralized configuration

**Cons:**

- Unknown time investment
- Traefik behavior not working as documented
- More complex architecture
- Coupling to specific ingress controller

**Why Rejected:** Application-layer CORS is proven, simpler, and works reliably

### Alternative 2: Use Both Layers

**Pros:**

- Redundancy?

**Cons:**

- ❌ Causes duplicate header errors
- ❌ Violates industry best practice
- ❌ More complex to maintain
- ❌ Harder to debug

**Why Rejected:** Industry consensus is "choose one, not both"

### Alternative 3: Switch to NGINX Ingress

**Pros:**

- Different ingress controller might work better

**Cons:**

- ❌ Major infrastructure change
- ❌ NGINX has its own CORS issues
- ❌ Doesn't solve fundamental problem

**Why Rejected:** Not worth infrastructure disruption

## References

### Fastify Documentation

- [fastify-cors](https://github.com/fastify/fastify-cors) - Official Fastify CORS plugin
- [Fastify Async/Await](https://fastify.dev/docs/latest/Reference/Server/#async-await) - Plugin registration patterns

### Related PRs

- platform-backend PR #160: Add async createApp with @fastify/cors
- steady-manifests PR #40: Remove Traefik CORS middleware

### Best Practices

- [EdgeX Foundry CORS](https://docs.edgexfoundry.org/3.1/security/Ch-CORS-Settings/) - Choose one layer
- [OWASP CORS Security](https://owasp.org/www-community/attacks/CORS_OriginHeaderScrutiny) - Security guidelines

---

**Document Version:** 1.0
**Date:** 2025-11-13
**Author:** Platform Team
**Supersedes:** ADR-001
