# Steady Manifests

Kubernetes manifests for the Steady product, managed by ArgoCD.

## Structure

- `inference-backend/` - AI inference service manifests
- `platform-backend/` - Platform API service manifests

## Branches

- `stage` - Stage environment (auto-deployed from service `dev` branches)
- `prod` - Production environment (auto-deployed from service `master` branches)

## Workflow

Service CI updates manifests automatically:
- Service push to `dev` → CI updates `stage` branch
- Service push to `master` → CI updates `prod` branch
- ArgoCD syncs changes to cluster

## Security

- SealedSecrets for credentials
- Database CRDs declare required databases
- No manual edits - CI generated only
