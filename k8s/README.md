# Kubernetes

Configuration for the single-node kubeadm cluster on AldiServer.

| File | Applied with |
|---|---|
| [`metallb/ip-pool.yaml`](metallb/ip-pool.yaml) | `kubectl apply -f k8s/metallb/ip-pool.yaml` |
| [`traefik/values.yaml`](traefik/values.yaml) | `helm install traefik traefik/traefik --namespace traefik --create-namespace --version 42.2.0 -f k8s/traefik/values.yaml` |
| [`argocd/install/`](argocd/install/) | `kubectl apply -k k8s/argocd/install --server-side --force-conflicts` |
| [`argocd/cv-application.yaml`](argocd/cv-application.yaml) | `kubectl apply -f k8s/argocd/cv-application.yaml` (once, after ArgoCD is installed) |

These files are applied by hand. ArgoCD only syncs the CV manifests in [`CV_Aldi_Lukito/`](../CV_Aldi_Lukito/).
