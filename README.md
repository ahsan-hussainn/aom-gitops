# aom-gitops

GitOps source of truth for the AOM platform. ArgoCD reconciles cluster
state against the manifests in this repo.

## Layout

```
apps/
  dagster/
    application.yaml   # ArgoCD Application CR — register this once, then ArgoCD owns the rest
    values.yaml        # Helm values overlay applied to the upstream dagster chart
```

## Workflow

1. Edit `apps/<app>/values.yaml` (or other manifests).
2. `git push origin main`.
3. ArgoCD detects the change and reconciles within ~3 minutes
   (faster if you click **Sync** in the UI).

Adding a new app: drop a new folder under `apps/` with its own
`application.yaml` and supporting manifests, then `kubectl apply -f`
that Application CR once. After that, all changes flow through git.
