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

## Implementation Summary

### Key Components

1. **Traefik CORS Middleware**: Created in `infrastructure/traefik-middleware/overlays/{stage,prod}/cors-middleware.yaml`
   - Defines allowed origins (environment-specific)
   - Configures methods, headers, credentials, cache time
   - Uses Traefik Headers Middleware CRD

2. **IngressRoute Updates**: Modified in `applications/platform-backend/overlays/{stage,prod}/ingressroute.yaml`
   - Added middleware reference to routes
   - CORS headers applied at ingress layer

3. **Application Changes**: Remove from `platform-backend` repository
   - Remove `@fastify/cors` plugin registration
   - Remove CORS configuration from config files
   - Remove `CORS_ORIGINS` environment variable
   - Uninstall `@fastify/cors` package

### Testing Strategy

Verify CORS functionality:

- Allowed origins receive proper CORS headers
- OPTIONS preflight requests work
- Credentials (cookies) are supported
- Unauthorized origins are rejected
- No duplicate headers appear

### Rollback Plan

If issues arise, revert IngressRoute middleware reference and redeploy application with `@fastify/cors` re-enabled

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

1. **Never use wildcards with credentials** - Explicit origin lists only
2. **HTTPS-only origins** - No HTTP in production
3. **Appropriate cache times** - 24 hours balances performance and security
4. **Minimal allowed headers** - Only include headers actually used

See [OWASP CORS Security](https://owasp.org/www-community/attacks/CORS_OriginHeaderScrutiny) for detailed guidance

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
