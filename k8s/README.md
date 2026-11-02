# Kubernetes

Configuration for the single-node kubeadm cluster on AldiServer.

| File | Applied with |
|---|---|
| [`metallb/ip-pool.yaml`](metallb/ip-pool.yaml) | `kubectl apply -f k8s/metallb/ip-pool.yaml` |
| [`traefik/values.yaml`](traefik/values.yaml) | `helm install traefik traefik/traefik --namespace traefik --create-namespace --version 42.2.0 -f k8s/traefik/values.yaml` |
| [`argocd/install/`](argocd/install/) | `kubectl apply -k k8s/argocd/install --server-side --force-conflicts` |
| [`argocd/cv-application.yaml`](argocd/cv-application.yaml) | `kubectl apply -f k8s/argocd/cv-application.yaml` (once, after ArgoCD is installed) |
| [`argocd/catalog-service-application.yaml`](argocd/catalog-service-application.yaml) | `kubectl apply -f k8s/argocd/catalog-service-application.yaml` (once, after [its database exists](../apps/ecommerce/README.md#add-a-database-for-catalog-service) and its first image is built) |
| [`cloudflared/`](cloudflared/) | `kubectl apply -f k8s/cloudflared/namespace.yaml`, create the `cloudflared-token` Secret, then `kubectl apply -f k8s/cloudflared/deployment.yaml` |
| [`argocd/search-service-application.yaml`](argocd/search-service-application.yaml) | `kubectl apply -f k8s/argocd/search-service-application.yaml` (once, after [its users exist](../apps/ecommerce/README.md#add-the-users-for-search-service) and its first image is built) |
| [`argocd/user-service-application.yaml`](argocd/user-service-application.yaml) | `kubectl apply -f k8s/argocd/user-service-application.yaml` (once, after [its database and key exist](../apps/ecommerce/README.md#add-a-database-and-key-for-user-service) and its first image is built) |
| [`argocd/web-application.yaml`](argocd/web-application.yaml) | `kubectl apply -f k8s/argocd/web-application.yaml` (once, after its first image is built) |
| [`argocd/order-service-application.yaml`](argocd/order-service-application.yaml) | `kubectl apply -f k8s/argocd/order-service-application.yaml` (once, after [its database and users exist](../apps/ecommerce/README.md#add-the-databases-and-users-for-the-order-saga) and its first image is built) |
| [`argocd/payment-service-application.yaml`](argocd/payment-service-application.yaml) | `kubectl apply -f k8s/argocd/payment-service-application.yaml` (once, after the same) |
| [`argocd/notification-service-application.yaml`](argocd/notification-service-application.yaml) | `kubectl apply -f k8s/argocd/notification-service-application.yaml` (once, after [its database, user and tokens exist](../apps/ecommerce/README.md#add-the-database-user-and-tokens-for-notification-service) and its first image is built) |
| [`argocd/realtime-service-application.yaml`](argocd/realtime-service-application.yaml) | `kubectl apply -f k8s/argocd/realtime-service-application.yaml` (once, after [NATS and its Secrets exist](../apps/ecommerce/README.md#install-it-with-its-passwords) and its first image is built) |

These files are applied by hand. ArgoCD syncs the application manifests in [`apps/`](../apps/). The online shop has its own folder, [`apps/ecommerce/`](../apps/ecommerce/).
