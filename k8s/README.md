# Kubernetes

Configuration for the single-node kubeadm cluster on AldiServer.

| File | Applied with |
|---|---|
| [`metallb/ip-pool.yaml`](metallb/ip-pool.yaml) | `kubectl apply -f k8s/metallb/ip-pool.yaml` |
| [`traefik/values.yaml`](traefik/values.yaml) | `helm install traefik traefik/traefik --namespace traefik --create-namespace --version 42.2.0 -f k8s/traefik/values.yaml` |
| [`argocd/install/`](argocd/install/) | `kubectl apply -k k8s/argocd/install --server-side --force-conflicts` |
| [`argocd/cv-application.yaml`](argocd/cv-application.yaml) | `kubectl apply -f k8s/argocd/cv-application.yaml` (once, after ArgoCD is installed) |
| [`argocd/ecommerce-application.yaml`](argocd/ecommerce-application.yaml) | `kubectl apply -f k8s/argocd/ecommerce-application.yaml` (once, after [the shop's namespaces and Secrets exist](../apps/ecommerce/README.md#deploy-the-services)) |
| [`argocd/studio-application.yaml`](argocd/studio-application.yaml) | `kubectl apply -f k8s/argocd/studio-application.yaml` (once, after [the studio's namespace and Secrets exist](../apps/studio-video/README.md#first-setup)) |
| [`coredns/configmap.yaml`](coredns/configmap.yaml) | `kubectl apply -f k8s/coredns/configmap.yaml`. It replaces kubeadm's CoreDNS settings, so check it again after a `kubeadm upgrade` |
| [`cloudflared/`](cloudflared/) | `kubectl apply -f k8s/cloudflared/namespace.yaml`, create the `cloudflared-token` Secret, then `kubectl apply -f k8s/cloudflared/deployment.yaml` |

These files are applied by hand. ArgoCD has four Applications, kept apart: `cv` for [`apps/cvaldi/`](../apps/cvaldi/), `ecommerce` for every microservice and tool in [`apps/ecommerce/`](../apps/ecommerce/), `saham` for the stock-signal project in [`apps/saham/`](../apps/saham/), and `studio` for Studio Video in [`apps/studio-video/`](../apps/studio-video/). The monitoring in `ecommerce` watches the first three.
