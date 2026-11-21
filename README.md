# devops-marathon-gitops

Everything that runs in the [devops-marathon](https://github.com/barisMarathon/devops-marathon-infra) k3s cluster, managed by **Argo CD** with the app-of-apps pattern. Changes go through Git; nobody runs `kubectl apply` against the cluster except for the one-time bootstrap below.

## Layout

```
bootstrap/root-app.yaml   the only manifest applied by hand: points Argo CD at apps/
apps/                     one Argo CD Application per component (sync waves set the order)
manifests/                the components themselves (kustomize / plain YAML)
```

| Component | Source | Wave |
|---|---|---|
| Argo CD (self-managed) | upstream `install.yaml` v3.5.6, trimmed with kustomize (no Dex, notifications, ApplicationSet) | -1 |
| cert-manager | Helm chart `jetstack/cert-manager` v1.21.3 | 0 |
| ClusterIssuers | Let's Encrypt staging + prod, DNS-01 via Cloudflare | 1 |
| Traefik config | k3s-bundled Traefik: production wildcard cert as default TLS, HTTP→HTTPS redirect | 2 |
| hello | nginx (unprivileged) demo page at https://hello.devopsdelisi.fun | 3 |

## Bootstrap (once per cluster)

```bash
export KUBECONFIG=~/.kube/devops-marathon.yaml

# 1. Install Argo CD from the same manifests it will later manage itself with
kubectl apply -k manifests/argocd --server-side --force-conflicts
kubectl -n argocd rollout status deploy/argocd-server deploy/argocd-repo-server
kubectl -n argocd rollout status statefulset/argocd-application-controller

# 2. Cloudflare API token for DNS-01 (Edit zone DNS, scoped to devopsdelisi.fun only)
kubectl create namespace cert-manager
kubectl -n cert-manager create secret generic cloudflare-api-token \
  --from-literal=api-token='<CLOUDFLARE_API_TOKEN>'

# 3. Hand control over to Git
kubectl apply -f bootstrap/root-app.yaml
```

## Argo CD UI (over Tailscale, not exposed publicly)

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
# https://localhost:8080  user: admin
```

## Verify

```bash
kubectl -n argocd get applications          # all Synced / Healthy
kubectl get clusterissuers                   # both Ready
kubectl -n kube-system get certificate       # wildcard-devopsdelisi Ready
curl -I http://hello.devopsdelisi.fun        # 308 -> https
curl -s https://hello.devopsdelisi.fun | head
```

## Secrets

No secrets are stored in this repo. The Cloudflare token is created by hand for now; encrypted secrets (SOPS or Sealed Secrets) come in a later phase.
