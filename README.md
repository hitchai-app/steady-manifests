# Steady Manifests

Kubernetes manifests for the Steady product, managed by ArgoCD.

## Structure

- `inference-backend/` - AI inference service manifests
- `platform-backend/` - Platform API service manifests
- `miniapp/` - Telegram Mini App frontend manifests
- `dev-ui/` - Development UI frontend manifests

## Branches

- `master` - Single branch with environment-specific overlays (`overlays/stage/` and `overlays/prod/`)

## Workflow

Service CI updates manifests automatically:
- Service push to `dev` → CI updates stage overlay image tags on `master`
- Service push to `master` → CI updates prod overlay image tags on `master`
- ArgoCD syncs changes from `master` to appropriate cluster

## Security

- SealedSecrets for credentials
- Database CRDs declare required databases
- No manual edits - CI generated only
