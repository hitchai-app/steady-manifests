# ADR-001: Use Traefik IngressRoute for Application-Layer CORS

## Status

Accepted

## Context

Platform-backend implements CORS at the application layer using `@fastify/cors` (Fastify framework). When testing directly against the Kubernetes service, CORS headers are correctly set by the application:

```bash
# Direct pod access - CORS headers present ✅
curl -i http://localhost:13000/health -H "Origin: https://app-stage.steady.ops.last-try.org"
# Response includes:
# access-control-allow-origin: https://app-stage.steady.ops.last-try.org
# access-control-allow-credentials: true
```

However, when requests pass through the Traefik ingress controller using standard Kubernetes Ingress resource, CORS headers are stripped:

```bash
# Through ingress - CORS headers missing ❌
curl -i https://api-stage.steady.ops.last-try.org/health -H "Origin: https://app-stage.steady.ops.last-try.org"
# No Access-Control-* headers in response
```

### Current Configuration

**Application (Working):**

- CORS plugin: `@fastify/cors` registered in src/app.ts
- Configuration: `CORS_ORIGINS` environment variable with comma-separated allowed origins
- Credentials support enabled

**Infrastructure (Issue):**

- Standard Kubernetes Ingress resource with Traefik ingress class
- TLS termination at ingress
- No CORS middleware configured
- Headers stripped somewhere in the request pipeline

### Industry Research

#### Where to Handle CORS

Research from 2024 shows two valid patterns:

1. **API Gateway Pattern (Centralized CORS)**
   - Best for: Multiple services with identical CORS policies
   - Pros: Single configuration point, handles preflight at edge
   - Cons: Less flexibility, harder to test, service couples to infrastructure

2. **Application Layer Pattern (Distributed CORS)**
   - Best for: Microservices with varying CORS requirements
   - Pros: Service autonomy, fine-grained control, testable in code
   - Cons: Per-service configuration

**Reference:** AWS API Gateway Documentation states "API Gateway supports both models: CORS headers can be added by an ingress gateway or set explicitly by the applications" ([AWS CORS Docs](https://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-cors.html))

#### Critical Rule: Never Both

Configuring CORS at both gateway and application layers causes duplicate `Access-Control-Allow-Origin` headers, which browsers reject as invalid.

**References:**

- [Stack Overflow: Duplicate CORS Headers](https://stackoverflow.com/questions/62495002/microservice-architecture-cors-configuration-results-in-access-control-allow-o)
- [Traefik Issue #5567](https://github.com/traefik/traefik/issues/5567): "The header inserted by Traefik does not replace the header sent by the application, resulting in invalid CORS configuration"

### Known Issues with Ingress Controllers

#### Traefik with Standard Ingress

**Issue:** Strips application-set CORS headers
**References:**

- [Traefik Issue #10432](https://github.com/traefik/traefik/issues/10432): "Traefik is removing Access-Control-Allow-Origin headers from the service response" (Feb 2024)
- [Traefik Issue #5567](https://github.com/traefik/traefik/issues/5567): "Application sent CORS headers are either not overwritten, or blocked completely" (Oct 2019)

**Note:** Traefik maintainers could not reproduce in standard configurations, suggesting the issue is environment-specific or related to TLS termination.

#### NGINX Ingress Alternative

Researched as potential alternative but found to have **worse** behavior for application-layer CORS:

**Issue:** Blocks OPTIONS preflight requests with 405 Method Not Allowed
**References:**

- [GitHub Issue #4393](https://github.com/kubernetes/ingress-nginx/issues/4393): "CORS OPTION request is intercepted by nginx and is not passed through to our application. Nginx returns a 405"
- [ServerFault](https://serverfault.com/questions/1104905/disable-cors-using-kubernetes-ingress): "These CORS settings only function if headers aren't already set by your application"

**Comparison:**

| Aspect                   | Traefik                          | NGINX Ingress                 |
| ------------------------ | -------------------------------- | ----------------------------- |
| **Issue**                | Strips headers                   | Blocks OPTIONS (405)          |
| **Solution**             | IngressRoute CRD                 | Configuration snippets        |
| **Complexity**           | Medium                           | High                          |
| **CORS Spec Compliance** | Better (distinguishes preflight) | Worse (unconditional headers) |

**Reference:** [ServerFault: Gateway CORS Comparison](https://serverfault.com/questions/1195815/difference-in-cors-handling-between-gateway-api-and-nginx-ingress) - "NGINX responds with all configured headers unconditionally, while Traefik handles headers differently between pre-flight and regular requests"

## Decision

We will **switch from standard Kubernetes Ingress to Traefik IngressRoute CRD** while **keeping CORS implementation at the application layer**.

### Rationale

1. **Preserves Working Architecture**
   - Application-layer CORS is already implemented, tested, and working
   - No code changes required in platform-backend or other services
   - Maintains microservices service autonomy principle

2. **Traefik Native Resource**
   - IngressRoute is Traefik's native CRD with better integration than Kubernetes Ingress
   - Community reports indicate fewer header manipulation issues with IngressRoute
   - Better control over routing behavior

3. **Avoids Alternative Problems**
   - NGINX Ingress has worse issues (blocks OPTIONS requests)
   - Moving CORS to ingress middleware contradicts microservices best practices
   - Configuration snippets (workaround) are more complex than IngressRoute

4. **Maintains Best Practices**
   - CORS configuration stays with the service (DDD principle)
   - Type-safe configuration in TypeScript with Zod validation
   - Testable in integration tests
   - Easy to have different CORS policies per service/endpoint

5. **Future-Proof**
   - Traefik's native CRD is better supported than Kubernetes Ingress
   - Enables use of Traefik-specific features (middleware, routing)
   - Aligns with Traefik's recommended patterns

## Consequences

### Positive

- ✅ Application continues to own its CORS policy (service autonomy)
- ✅ No changes to application code required
- ✅ Type-safe CORS configuration in TypeScript
- ✅ CORS behavior testable in integration tests
- ✅ Better Traefik integration and control
- ✅ Flexibility for per-service/per-endpoint CORS policies
- ✅ Avoids known issues with both standard Ingress and NGINX

### Negative

- ⚠️ Requires manifest changes in steady-manifests repository
- ⚠️ Deviation from standard Kubernetes Ingress (though using Traefik-native resource)
- ⚠️ Team needs to understand IngressRoute in addition to standard Ingress
- ⚠️ Migration required for other services if they face similar issues

### Neutral

- IngressRoute is well-documented and widely used in Traefik deployments
- ArgoCD handles IngressRoute CRDs the same as standard Ingress resources
- Rollback path: can revert to Ingress if IngressRoute doesn't solve the issue

## Implementation

### Phase 1: Platform-Backend (Stage Environment)

**Create IngressRoute:**

```yaml
# applications/platform-backend/base/ingressroute.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: platform-backend
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`api-stage.steady.ops.last-try.org`)
      kind: Rule
      services:
        - name: platform-backend
          port: 3000
  tls:
    secretName: platform-backend-tls
```

**Update Kustomization:**

```yaml
# applications/platform-backend/base/kustomization.yaml
resources:
  - deployment.yaml
  - service.yaml
  - database.yaml
  - migration-job.yaml
  - ingressroute.yaml # Changed from ingress.yaml
```

**No Application Changes Required:**

- Keep existing `@fastify/cors` configuration
- Keep existing `CORS_ORIGINS` environment variable
- No code modifications

### Phase 2: Testing

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
# Expected: No access-control-allow-origin header (or proper rejection)
```

### Phase 3: Rollout

1. Deploy to stage environment via ArgoCD
2. Verify CORS headers work correctly
3. Monitor for 24-48 hours
4. Roll out to production environment
5. Document pattern for other services

### Rollback Plan

If IngressRoute doesn't solve the issue:

1. Revert kustomization.yaml to use ingress.yaml
2. Consider Option B: Move CORS to Traefik middleware (less preferred)
3. Investigate other root causes (global middleware, Traefik config)

## Alternatives Considered

### Alternative 1: Move CORS to Traefik Middleware

**Approach:** Configure CORS at ingress layer, remove from application

**Implementation:**

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: cors-headers
  namespace: traefik
spec:
  headers:
    accessControlAllowOriginListRegex:
      - "https://app-stage\\.steady\\.ops\\.last-try\\.org"
      - "https://miniapp-stage\\.steady\\.ops\\.last-try\\.org"
    accessControlAllowMethods:
      - "GET"
      - "POST"
      - "PUT"
      - "DELETE"
      - "PATCH"
      - "OPTIONS"
    accessControlAllowCredentials: true
```

**Why Rejected:**

- ❌ Violates service autonomy principle
- ❌ Requires removing working application code
- ❌ CORS config separated from application logic
- ❌ Cannot test CORS in unit/integration tests
- ❌ Less flexible for per-endpoint policies
- ❌ Known bugs with Traefik middleware ordering

**Reference:** [Traefik Community](https://stackoverflow.com/questions/56644462/can-we-set-priority-for-the-middlewares-in-traefik-v2) - Reports of issues with middleware priority when combining CORS with other middleware

### Alternative 2: Switch to NGINX Ingress

**Why Rejected:**

- ❌ Worse behavior: blocks OPTIONS requests with 405
- ❌ Requires complex configuration snippets for pass-through
- ❌ Infrastructure migration overhead
- ❌ Team already familiar with Traefik

### Alternative 3: Configure Traefik Global Settings

**Why Rejected:**

- ❌ Diagnostic investigation showed no global middleware interfering
- ❌ No evidence of misconfiguration in Traefik deployment
- ❌ Issue appears specific to standard Ingress resource behavior

## Security Considerations

### CORS Security Best Practices

1. **Never use wildcards with credentials:**

   ```yaml
   # ❌ INSECURE
   Access-Control-Allow-Origin: "*"
   Access-Control-Allow-Credentials: true

   # ✅ SECURE (our implementation)
   Access-Control-Allow-Origin: https://app-stage.steady.ops.last-try.org
   Access-Control-Allow-Credentials: true
   ```

2. **Explicit origin validation:**
   - Our config: Zod-validated environment variable with explicit domain list
   - No regex wildcards in production
   - HTTPS-only origins enforced

3. **Appropriate cache times:**
   - `Access-Control-Max-Age: 100` (Fastify default)
   - Balances performance with security

**Reference:** [OWASP CORS Security](https://owasp.org/www-community/attacks/CORS_OriginHeaderScrutiny)

## References

### Official Documentation

- [Traefik IngressRoute Documentation](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/)
- [Traefik Headers Middleware](https://doc.traefik.io/traefik/middlewares/http/headers/)
- [AWS API Gateway CORS](https://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-cors.html)
- [MDN: Cross-Origin Resource Sharing](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)

### Known Issues

- [Traefik #10432: Headers removed from service response](https://github.com/traefik/traefik/issues/10432) (Feb 2024)
- [Traefik #5567: Application headers not overwritten](https://github.com/traefik/traefik/issues/5567) (Oct 2019)
- [NGINX Ingress #4393: CORS OPTIONS blocked](https://github.com/kubernetes/ingress-nginx/issues/4393)

### Best Practices

- [Medium: CORS on Service Meshes](https://medium.com/@stanislavbabenko/cors-on-service-mesh-your-pattern-is-wrong-yes-that-means-you-senior-solutions-architect-63b8d0b8a804)
- [Stack Overflow: Traefik vs NGINX CORS](https://serverfault.com/questions/1195815/difference-in-cors-handling-between-gateway-api-and-nginx-ingress)

---

**Document Version:** 1.0
**Date:** 2025-11-12
**Author:** Platform Team
**Reviewers:** TBD
