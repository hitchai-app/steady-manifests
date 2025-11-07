# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Kubernetes manifest repository for the Steady product, managed by ArgoCD. It contains GitOps-style deployment configurations that are automatically updated by service CI pipelines. **Manual edits to deployment manifests are not recommended** - changes should come from the service repositories' CI systems.

## Architecture

### Branch Strategy

- `master` - Single branch managing both stage and prod environments via Kustomize overlays
- Service CI updates image tags in environment-specific overlays (`overlays/stage/` and `overlays/prod/`)

### Repository Structure

```
applications/           # Application deployments
  dev-ui/              # Development UI frontend
  inference-backend/   # AI inference service
  platform-backend/    # Platform API service

infrastructure/        # Infrastructure components (base + overlays)
  postgres/            # CloudNativePG PostgreSQL cluster
  valkey/              # Redis-compatible key-value store
  centrifugo/          # Real-time messaging server
  litellm/             # LiteLLM proxy for AI models

secrets/               # SealedSecrets (encrypted credentials)
```

### Kustomize Structure

All infrastructure components follow a base/overlays pattern:
- `base/` - Common configuration
- `overlays/prod/` - Production-specific configuration
- `overlays/stage/` - Stage-specific configuration

Root `kustomization.yaml` assembles all components for the environment.

## Key Technologies

### CloudNativePG (CNPG)
PostgreSQL is managed by the CloudNativePG operator:
- **Cluster**: `postgres` (single instance in prod)
- **Databases**: Declared via `Database` CRD in application directories
- **Users**: Managed via `spec.managed.roles` in cluster.yaml
- **Credentials**: SealedSecrets with connection URIs

When adding a new service requiring Postgres:
1. Create `database.yaml` with `Database` CRD in application directory
2. Create sealed secret with connection URI (format: `postgres-{service}-user`)
3. Add managed role to `infrastructure/postgres/base/cluster.yaml`

### SealedSecrets
Credentials are encrypted using Bitnami SealedSecrets:
- Encrypted secrets committed to git in `secrets/`
- Controller decrypts them in-cluster
- Never commit plain Secrets - always seal them first

### ArgoCD Sync Waves
Deployments use annotations to control sync order:
- `argocd.argoproj.io/sync-wave: "1"` - Migration jobs
- `argocd.argoproj.io/sync-wave: "2"` - Application deployments
- No annotation (wave "0") - Infrastructure and databases

## CI/CD Integration

### Automated Image Updates

The `.github/workflows/update-image.yml` workflow responds to `repository_dispatch` events from service CI:

```bash
# Service CI triggers update via:
curl -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  -d '{"event_type": "image_update", "client_payload": {...}}' \
  https://api.github.com/repos/{org}/steady-prod/dispatches
```

The workflow:
1. Checks out the master branch
2. Updates `image:` tags in environment-specific overlay kustomizations
3. Commits and pushes changes to master
4. ArgoCD detects commit and syncs to appropriate cluster

### Manual Image Updates

If you need to manually update an image (not recommended):

```bash
# Update stage image tag
sed -i "s|newTag: .*|newTag: stage-new-tag|" \
  applications/platform-backend/overlays/stage/kustomization.yaml

# Update prod image tag
sed -i "s|newTag: .*|newTag: prod-new-tag|" \
  applications/platform-backend/overlays/prod/kustomization.yaml

# Commit changes
git add applications/platform-backend/
git commit -m "chore(platform-backend): update image to new-tag"
git push
```

## Common Operations

### Validating Manifests

```bash
# Validate Kubernetes YAML syntax
kubectl apply --dry-run=client -f applications/platform-backend/

# Build and validate kustomization
kubectl kustomize . --enable-helm

# Validate specific component
kubectl kustomize infrastructure/postgres/overlays/prod
```

### Working with SealedSecrets

```bash
# Seal a new secret (requires kubeseal CLI and access to cluster)
kubeseal --format yaml < secret.yaml > sealed-secret.yaml

# View what's in a sealed secret (without decrypting)
kubectl get sealedsecret ghcr-pull-secret -n steady-prod -o yaml
```

### Checking Database CRDs

```bash
# View all Database resources
kubectl get database -n steady-prod

# View Database details
kubectl get database platform-db -n steady-prod -o yaml
```

## Application Deployment Patterns

### Standard Application Components

Each application typically includes:
- `deployment.yaml` - Main deployment with env vars, secrets, health checks
- `service.yaml` - ClusterIP service
- `kustomization.yaml` - Kustomize resource list
- `database.yaml` - Database CRD (if uses Postgres)
- `migration-job.yaml` - Optional pre-deployment migration Job
- `ingress.yaml` - Optional ingress for external access

### Environment Variables Pattern

Services receive configuration via environment variables:
- Database: `DATABASE_URL` from sealed secret
- Redis/Valkey: `REDIS_HOST`, `REDIS_PORT` (hardcoded service names)
- Centrifugo: `CENTRIFUGO_URL`, `CENTRIFUGO_API_KEY`, `CENTRIFUGO_JWT_SECRET`
- Runtime: `NODE_ENV=production`

### Health Checks

All services should implement:
- `GET /health` - Liveness probe (basic health)
- `GET /ready` - Readiness probe (dependencies ready)

## Important Constraints

1. **CI-Generated Manifests**: This repo is primarily maintained by CI. Manual changes to `image:` fields will be overwritten.

2. **No Direct Secret Management**: Never commit unencrypted secrets. Use SealedSecrets controller.

3. **Database Schema Migrations**: Migrations run as pre-sync Jobs (sync-wave "1"). They must be idempotent.

4. **Resource Limits**: All containers must specify resource requests/limits for proper scheduling.

5. **Namespace**: All resources deploy to `steady-prod` namespace (defined in root kustomization).

## Troubleshooting

### Image Pull Issues
Check `ghcr-pull-secret` SealedSecret exists and is valid.

### Database Connection Issues
- Verify `Database` CRD exists and has `ensure: present`
- Check sealed secret contains valid `uri` key
- Confirm user exists in `postgres` cluster's `spec.managed.roles`

### Migration Job Failures
- Check Job logs: `kubectl logs -n steady-prod job/platform-backend-migration`
- Verify `backoffLimit` hasn't been exceeded
- Ensure migration is idempotent

### ArgoCD Sync Failures
- Check sync-wave ordering (migrations before deployments)
- Verify all referenced secrets exist
- Check resource dependencies (Database CRD before Deployment)
