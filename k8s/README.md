# Kubernetes

Configuration for the single-node kubeadm cluster on AldiServer.

| File | Applied with |
|---|---|
| [`metallb/ip-pool.yaml`](metallb/ip-pool.yaml) | `kubectl apply -f k8s/metallb/ip-pool.yaml` |
| [`traefik/values.yaml`](traefik/values.yaml) | `helm install traefik traefik/traefik --namespace traefik --create-namespace --version 42.2.0 -f k8s/traefik/values.yaml` |
| [`argocd/install/`](argocd/install/) | `kubectl apply -k k8s/argocd/install --server-side --force-conflicts` |
| [`argocd/cv-application.yaml`](argocd/cv-application.yaml) | `kubectl apply -f k8s/argocd/cv-application.yaml` (once, after ArgoCD is installed) |
| [`argocd/ecommerce-application.yaml`](argocd/ecommerce-application.yaml) | `kubectl apply -f k8s/argocd/ecommerce-application.yaml` (once, after [the shop's namespaces and Secrets exist](../apps/ecommerce/README.md#deploy-the-services)) |
| [`cloudflared/`](cloudflared/) | `kubectl apply -f k8s/cloudflared/namespace.yaml`, create the `cloudflared-token` Secret, then `kubectl apply -f k8s/cloudflared/deployment.yaml` |

These files are applied by hand. ArgoCD has two Applications: `cv` for [`apps/cvaldi/`](../apps/cvaldi/), and `ecommerce` for every microservice in [`apps/ecommerce/`](../apps/ecommerce/).
