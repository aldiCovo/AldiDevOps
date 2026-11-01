# Kubernetes

Configuration for the single-node kubeadm cluster on AldiServer.

| File | Applied with |
|---|---|
| [`metallb/ip-pool.yaml`](metallb/ip-pool.yaml) | `kubectl apply -f k8s/metallb/ip-pool.yaml` |
| [`traefik/values.yaml`](traefik/values.yaml) | `helm install traefik traefik/traefik --namespace traefik --create-namespace --version 42.2.0 -f k8s/traefik/values.yaml` |
| [`argocd/install/`](argocd/install/) | `kubectl apply -k k8s/argocd/install --server-side --force-conflicts` |
| [`argocd/cv-application.yaml`](argocd/cv-application.yaml) | `kubectl apply -f k8s/argocd/cv-application.yaml` (once, after ArgoCD is installed) |
| [`argocd/catalog-service-application.yaml`](argocd/catalog-service-application.yaml) | `kubectl apply -f k8s/argocd/catalog-service-application.yaml` (once, after [its database exists](#add-a-database-for-catalog-service) and its first image is built) |
| [`cloudflared/`](cloudflared/) | `kubectl apply -f k8s/cloudflared/namespace.yaml`, create the `cloudflared-token` Secret, then `kubectl apply -f k8s/cloudflared/deployment.yaml` |
| [`postgresql/`](postgresql/) | `kubectl apply -f k8s/postgresql/namespace.yaml`, create the `postgresql-superuser` Secret as below, then `kubectl apply -f k8s/postgresql/` |
| [`rabbitmq/`](rabbitmq/) | `kubectl apply -f k8s/rabbitmq/namespace.yaml`, create the `rabbitmq-admin` Secret as below, then `kubectl apply -f k8s/rabbitmq/` |

These files are applied by hand. ArgoCD syncs the application manifests in [`apps/`](../apps/).

## PostgreSQL

One instance serves every application, each with its own database and role. The data is in `/var/lib/aldi/postgresql` on AldiServer and has no backup yet.

Create the superuser Secret, once, before the first `kubectl apply -f k8s/postgresql/`:

```bash
kubectl create secret generic postgresql-superuser -n postgresql \
  --from-literal=password="$(openssl rand -hex 24)"
```

### Add a database for catalog-service

Run this once, on AldiServer, after the `postgresql-0` Pod is ready:

```bash
PASSWORD=$(openssl rand -hex 24)
kubectl exec -i -n postgresql postgresql-0 -- psql -U postgres -v ON_ERROR_STOP=1 -v password="$PASSWORD" <<'SQL'
CREATE ROLE catalog LOGIN PASSWORD :'password';
CREATE DATABASE catalog OWNER catalog;
SQL
kubectl create namespace catalog
kubectl create secret generic catalog-database -n catalog \
  --from-literal=url="postgres://catalog:$PASSWORD@postgresql.postgresql.svc:5432/catalog?sslmode=disable"
```

## RabbitMQ

One broker carries the events between applications. The data is in `/var/lib/aldi/rabbitmq` on AldiServer. It is only reachable inside the cluster, at `rabbitmq.rabbitmq.svc:5672`.

Create the admin Secret, once, before the first `kubectl apply -f k8s/rabbitmq/`. RabbitMQ only reads this password when its data folder is empty, so changing the Secret later doesn't change the password:

```bash
kubectl create secret generic rabbitmq-admin -n rabbitmq \
  --from-literal=password="$(openssl rand -hex 24)"
```

To open the management UI, run this on AldiServer, then browse to `http://localhost:15672` through an SSH tunnel and log in as `admin`:

```bash
kubectl port-forward -n rabbitmq svc/rabbitmq 15672:15672
kubectl get secret rabbitmq-admin -n rabbitmq -o jsonpath='{.data.password}' | base64 -d; echo
```
