# gitops-infra

Centralized GitOps repository for the UpCloud Kubernetes cluster. ArgoCD watches this repo and automatically applies changes to the cluster.

## Structure

```
gitops-infra/
├── root-app.yaml          # Root ArgoCD Application (App of Apps) — apply manually once
├── argocd-apps/           # One ArgoCD Application per service — managed by root-app
└── apps/                  # Kubernetes manifests per service
    ├── ai-whatif/         # AI What-If service (React frontend + R/Plumber backend)
    └── cluster-issuer/    # Let's Encrypt ClusterIssuer for SSL certificates
```

## Adding a new service

1. Create `apps/<service-name>/` with manifests (namespace, deployment, service, ingress)
2. Create `argocd-apps/<service-name>.yaml` pointing to that folder
3. Commit — ArgoCD deploys automatically

## Secrets

Secrets are never committed. Apply them manually:
```bash
kubectl apply -f secret.yaml -n <namespace>
```

## Cluster prerequisites

- ingress-nginx
- cert-manager
- ArgoCD
