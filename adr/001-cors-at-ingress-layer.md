# ADR-001: Configure CORS at Ingress Layer Using Traefik Middleware

## Status

Accepted

## Context

Platform-backend API (`api-stage.steady.ops.last-try.org`) and frontend applications (`app-stage.steady.ops.last-try.org`, `miniapp-stage.steady.ops.last-try.org`) are on different domains, requiring Cross-Origin Resource Sharing (CORS) configuration.

### Previous Approach (Incorrect)

ADR-001 attempted to preserve application-layer CORS headers (`@fastify/cors`) through Traefik IngressRoute. This approach was based on:

- Assumption that IngressRoute would pass through application headers better than standard Ingress
- Microservices principle of "service autonomy"
- Keeping CORS configuration in application code

**This approach failed:**

- CORS headers were stripped by Traefik regardless of using Ingress or IngressRoute
- This is a well-documented issue across Traefik v2.x through v3.5+
- Configuring CORS at both layers (application + ingress) causes duplicate headers that browsers reject

### Industry Research (2024-2025)

Comprehensive research of Kubernetes CORS best practices revealed:

**Industry Standard:** Configure CORS at the **ingress/gateway layer**, NOT application layer.

**Evidence:**

- All major cloud providers (AWS EKS, GCP GKE, Azure AKS) use ingress-layer CORS
- EdgeX Foundry (CNCF project): _"Implementers should choose one or the other [gateway or microservice], not both"_
- Fastify documentation warns against wildcards and recommends ingress-layer configuration for production
- GitHub issues document duplicate header failures when mixing layers

**Why Ingress Layer:**

- ✅ Centralized policy management across all microservices
- ✅ Better performance (preflight requests handled at edge, don't hit backend)
- ✅ Separation of concerns (network security at ingress, business logic in app)
- ✅ Prevents duplicate header issues
- ✅ Consistency across services (no fragmented CORS rules)

## Decision

We will **configure CORS at the Traefik ingress layer using Headers Middleware** and **remove CORS from application code entirely**.

### Rationale

1. **Follows Industry Standard**
   - Aligns with AWS, GCP, Azure patterns
   - Matches CNCF best practices
   - Proven in production at scale

2. **Traefik Native Approach**
   - Traefik Headers Middleware is designed for this use case
   - Reusable across multiple IngressRoutes
   - Better than custom annotations or hacks

3. **Prevents Duplicate Header Issues**
   - Application will not set CORS headers
   - Only Traefik sets them
   - Eliminates most common CORS production failure

4. **Better Performance**
   - OPTIONS preflight requests handled by Traefik directly
   - Backend doesn't process preflight requests
   - Reduces latency for CORS requests

5. **Centralized Management**
   - Single place to update allowed origins
   - Kustomize overlays for environment-specific origins
   - Consistent policy across all services

## Consequences

### Positive

- ✅ CORS headers work reliably through Traefik
- ✅ Centralized CORS policy management
- ✅ Better performance (edge preflight handling)
- ✅ Eliminates duplicate header issues
- ✅ Follows industry best practices
- ✅ Easier to audit and maintain security policies
- ✅ Consistent across all microservices

### Negative

- ⚠️ CORS configuration separated from application code
- ⚠️ Changes require manifest updates (not just app code)
- ⚠️ Deviates from "service autonomy" principle
- ⚠️ Platform-backend requires code changes to remove @fastify/cors

### Neutral

- CORS policy is now infrastructure configuration
- Easier to see security policies in one place (manifest repo)
- Standard practice in production Kubernetes

## Implementation

### Phase 1: Create Traefik CORS Middleware

**Stage Environment:**

```yaml
# infrastructure/traefik-middleware/overlays/stage/cors-middleware.yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: cors-headers
  namespace: steady-stage
spec:
  headers:
    accessControlAllowOriginList:
      - "https://app-stage.steady.ops.last-try.org"
      - "https://miniapp-stage.steady.ops.last-try.org"
    accessControlAllowMethods:
      - "GET"
      - "POST"
      - "PUT"
      - "DELETE"
      - "PATCH"
      - "OPTIONS"
    accessControlAllowHeaders:
      - "Content-Type"
      - "Authorization"
      - "X-Requested-With"
      - "X-CSRF-Token"
    accessControlAllowCredentials: true
    accessControlMaxAge: 86400
```

**Production Environment:**

```yaml
# infrastructure/traefik-middleware/overlays/prod/cors-middleware.yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: cors-headers
  namespace: steady-prod
spec:
  headers:
    accessControlAllowOriginList:
      - "https://app.steady.ops.last-try.org"
      - "https://miniapp.steady.ops.last-try.org"
    accessControlAllowMethods:
      - "GET"
      - "POST"
      - "PUT"
      - "DELETE"
      - "PATCH"
      - "OPTIONS"
    accessControlAllowHeaders:
      - "Content-Type"
      - "Authorization"
      - "X-Requested-With"
      - "X-CSRF-Token"
    accessControlAllowCredentials: true
    accessControlMaxAge: 86400
```

### Phase 2: Update IngressRoute to Use Middleware

**Stage:**

```yaml
# applications/platform-backend/overlays/stage/ingressroute.yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: platform-backend
  annotations:
    argocd.argoproj.io/sync-wave: "2"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`api-stage.steady.ops.last-try.org`)
      kind: Rule
      middlewares:
        - name: cors-headers
          namespace: steady-stage
      services:
        - name: platform-backend
          port: 3000
  tls:
    secretName: platform-backend-tls
```

**Production:**

```yaml
# applications/platform-backend/overlays/prod/ingressroute.yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: platform-backend
  annotations:
    argocd.argoproj.io/sync-wave: "2"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`api.steady.ops.last-try.org`)
      kind: Rule
      middlewares:
        - name: cors-headers
          namespace: steady-prod
      services:
        - name: platform-backend
          port: 3000
  tls:
    secretName: platform-backend-tls
```

### Phase 3: Remove CORS from Platform-Backend Application

**Changes to `platform-backend` repository:**

1. Remove `@fastify/cors` plugin registration:

```typescript
// src/app.ts - REMOVE:
// await app.register(cors, {
//   origin: config.cors.origins,
//   credentials: true,
// });
```

2. Remove CORS configuration:

```typescript
// src/config.ts - REMOVE:
// cors: {
//   origins: z.string().transform(s => s.split(',')),
// }
```

3. Remove environment variable from deployment:

```yaml
# steady-manifests: applications/platform-backend/base/deployment.yaml
# REMOVE:
# - name: CORS_ORIGINS
#   value: "..."
```

4. Uninstall package:

```bash
npm uninstall @fastify/cors
```

### Phase 4: Testing

**Test Plan:**

```bash
# 1. CORS headers present for allowed origin
curl -i https://api-stage.steady.ops.last-try.org/health \
  -H "Origin: https://app-stage.steady.ops.last-try.org"
# Expected: access-control-allow-origin: https://app-stage.steady.ops.last-try.org

# 2. OPTIONS preflight works
curl -i -X OPTIONS https://api-stage.steady.ops.last-try.org/health \
  -H "Origin: https://app-stage.steady.ops.last-try.org" \
  -H "Access-Control-Request-Method: GET"
# Expected: 200 OK with CORS headers

# 3. Credentials support
curl -i https://api-stage.steady.ops.last-try.org/health \
  -H "Origin: https://app-stage.steady.ops.last-try.org" \
  -H "Cookie: session=test"
# Expected: access-control-allow-credentials: true

# 4. Unauthorized origin rejected
curl -i https://api-stage.steady.ops.last-try.org/health \
  -H "Origin: https://evil.com"
# Expected: No access-control-allow-origin header

# 5. Verify no duplicate headers
curl -i https://api-stage.steady.ops.last-try.org/health \
  -H "Origin: https://app-stage.steady.ops.last-try.org" | \
  grep -i "access-control-allow-origin"
# Expected: Only ONE header line (not duplicated)
```

### Phase 5: Rollout

1. Deploy manifest changes to stage (Traefik middleware + IngressRoute update)
2. Verify CORS works correctly in stage
3. Deploy platform-backend code changes to stage (remove @fastify/cors)
4. Verify CORS still works after app deployment
5. Monitor for 24 hours
6. Roll out to production

### Rollback Plan

If issues arise:

1. Revert IngressRoute middleware reference
2. Re-enable @fastify/cors in platform-backend (redeploy app)
3. Investigate specific failure mode

## Alternatives Considered

### Alternative 1: Keep Application-Layer CORS (Original ADR-001)

**Why Rejected:**

- ❌ Doesn't work - Traefik strips headers (confirmed by testing and research)
- ❌ IngressRoute doesn't solve the issue (contrary to initial assumption)
- ❌ Goes against industry best practices
- ❌ All major cloud providers use ingress-layer CORS

### Alternative 2: Use Standard Kubernetes Ingress with NGINX

**Why Rejected:**

- ❌ Requires switching ingress controllers (major infrastructure change)
- ❌ NGINX Ingress has its own CORS issues (blocks OPTIONS with 405)
- ❌ Traefik Middleware is the proper solution for Traefik environments

### Alternative 3: Use Service Mesh (Istio) CORS Policy

**Why Rejected:**

- ❌ Overkill for just CORS (service mesh adds significant complexity)
- ❌ Not needed unless we need service mesh for other reasons
- ❌ Traefik Middleware is simpler and sufficient

## Security Considerations

### CORS Security Best Practices

1. **Never use wildcards with credentials:**

   ```yaml
   # ❌ INSECURE - Allows any origin
   accessControlAllowOriginList:
     - "*"

   # ✅ SECURE - Explicit origin list
   accessControlAllowOriginList:
     - "https://app-stage.steady.ops.last-try.org"
     - "https://miniapp-stage.steady.ops.last-try.org"
   ```

2. **Explicit origin validation:**
   - No regex wildcards in production
   - HTTPS-only origins enforced
   - Comma-separated list for multiple origins

3. **Appropriate cache times:**
   - `accessControlMaxAge: 86400` (24 hours)
   - Balances performance with security

4. **Minimal allowed headers:**
   - Only include headers actually used by frontend
   - Reduces attack surface

**Reference:** [OWASP CORS Security](https://owasp.org/www-community/attacks/CORS_OriginHeaderScrutiny)

## References

### Industry Best Practices

- [EdgeX Foundry Security](https://docs.edgexfoundry.org/3.1/security/Ch-CORS-Settings/) - CNCF project CORS guidelines
- [Fastify CORS Plugin](https://github.com/fastify/fastify-cors) - Production security warnings
- [AWS EKS Best Practices](https://aws.github.io/aws-eks-best-practices/) - Ingress-layer security
- [GCP GKE Security](https://cloud.google.com/kubernetes-engine/docs/concepts/ingress) - Ingress controller patterns

### Traefik Documentation

- [Traefik Headers Middleware](https://doc.traefik.io/traefik/middlewares/http/headers/)
- [Traefik IngressRoute](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/)
- [Traefik Kubernetes CRD](https://doc.traefik.io/traefik/reference/dynamic-configuration/kubernetes-crd/)

### Known Issues

- [Traefik #10432: Headers removed from service response](https://github.com/traefik/traefik/issues/10432)
- [Traefik #5567: Application headers not overwritten](https://github.com/traefik/traefik/issues/5567)
- [Traefik Community: CORS not working](https://community.traefik.io/t/cors-is-not-working-with-traefik/25906)

### Common Pitfalls

- [GitHub #2138: Duplicate CORS headers](https://github.com/expressjs/cors/issues/2138)
- [Stack Overflow: CORS configuration best practices](https://stackoverflow.com/questions/65343708/how-to-enable-cors-in-kubernetes-with-traefik)

---

**Document Version:** 1.0
**Date:** 2025-11-12
**Author:** Platform Team
**Supersedes:** ADR-001 (incorrect approach)
