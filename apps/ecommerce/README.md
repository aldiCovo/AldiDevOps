# Online shop

Everything for the shop, with one folder per service and per tool it uses.

| Folder | What it is | Applied with |
|---|---|---|
| [`catalog-service/`](catalog-service/) | Products and stock | ArgoCD |
| [`search-service/`](search-service/) | Product search | ArgoCD |
| [`user-service/`](user-service/) | Accounts and login | ArgoCD |
| [`order-service/`](order-service/) | Orders, and the saga that reserves stock and takes payment | ArgoCD |
| [`payment-service/`](payment-service/) | Payments, simulated for now | ArgoCD |
| [`realtime-service/`](realtime-service/) | Tells browsers when an order changes, over WebSockets | ArgoCD |
| [`notification-service/`](notification-service/) | Emails customers when their order finishes | ArgoCD |
| [`cart-service/`](cart-service/) | Keeps each customer's cart on their account | ArgoCD |
| [`web/`](web/) | The storefront at `prastika.aldiab.ai.id` | ArgoCD |
| [`admin-service/`](admin-service/) | Lets admins in and calls the other services' internal routes for them | ArgoCD, after the Secrets in [Add the tokens for the admin pages](#add-the-tokens-for-the-admin-pages) exist |
| [`web-admin/`](web-admin/) | The admin pages at `prastika.aldiab.ai.id/admin`, with their own login | ArgoCD |
| [`postgresql/`](postgresql/) | Database | ArgoCD, after the `postgresql-superuser` Secret below exists |
| [`rabbitmq/`](rabbitmq/) | Events between services | ArgoCD, after the `rabbitmq-admin` Secret below exists |
| [`elasticsearch/`](elasticsearch/) | Search index | ArgoCD, after the `elasticsearch-admin` Secret below exists |
| [`nats/`](nats/) | Signals that an order changed | ArgoCD, after the Secrets below exist |
| [`mailpit/`](mailpit/) | Catches the shop's emails, and delivers none | ArgoCD |
| [`monitoring/`](monitoring/) | Prometheus and Grafana at `grafana.aldiab.ai.id`, for the server, the cluster and the shop | ArgoCD, after the `grafana-admin` Secret below exists |

Everything here belongs to one Application in ArgoCD, `ecommerce`, which deploys what [`kustomization.yaml`](kustomization.yaml) lists. Secrets, and what's inside the tools, such as databases, users and vhosts, are created by hand, because this repo is public.

Tools sync before services. ArgoCD never deletes a tool's objects, including its volume and namespace, even if they leave Git or the Application is deleted. To remove a tool, delete its objects with `kubectl`. These folders can't be applied with `kubectl apply -f <folder>`, because each has a `kustomization.yaml`.

ArgoCD does apply a change to a tool as soon as it's merged, and most changes to a StatefulSet restart it. A restart of PostgreSQL or RabbitMQ interrupts every service, so check the diff in a PR first. PostgreSQL has no backup yet, and none of this protects against a failed disk.

## Deploy the services

Each service keeps its own folder, namespace and image tag. The namespaces are created by hand, because the Secrets below must exist before anything that uses them starts:

```bash
for NAMESPACE in postgresql rabbitmq elasticsearch nats mailpit monitoring catalog search user order payment realtime notification cart web admin web-admin; do
  kubectl create namespace "$NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -
done
kubectl apply -f k8s/argocd/ecommerce-application.yaml
```

To add a service, add its folder to `kustomization.yaml`. It needs no Application of its own.

To sync without waiting for ArgoCD's next check:

```bash
kubectl annotate application ecommerce -n argocd argocd.argoproj.io/refresh=normal --overwrite
```

## NATS

Core NATS, with no JetStream and no volume. It keeps no messages, so a restart only loses signals, and browsers catch up from the order status in PostgreSQL. RabbitMQ carries the events that must not be lost.

### Install it with its passwords

Run this once, on AldiServer, before the first sync. Without these Secrets, NATS and the services that use it wait until they exist:

```bash
ORDER_PASSWORD=$(openssl rand -hex 24)
REALTIME_PASSWORD=$(openssl rand -hex 24)
kubectl create secret generic nats-passwords -n nats \
  --from-literal=order="$ORDER_PASSWORD" --from-literal=realtime="$REALTIME_PASSWORD"
kubectl create secret generic order-nats -n order \
  --from-literal=url="nats://order:$ORDER_PASSWORD@nats.nats.svc:4222"
kubectl create secret generic realtime-nats -n realtime \
  --from-literal=url="nats://realtime:$REALTIME_PASSWORD@nats.nats.svc:4222"
unset ORDER_PASSWORD REALTIME_PASSWORD
```

## Mailpit

It catches every email and delivers none, so the shop can't send mail to strangers. It keeps the newest 500, and loses them when its Pod restarts.

To read them, run this on AldiServer, then browse to `http://localhost:8025` through an SSH tunnel. It has no Ingress and no login, because it shows customers' addresses, so never expose it:

```bash
kubectl port-forward -n mailpit svc/mailpit 8025:8025
```

### Add the database, user and tokens for notification-service

Run this once, on AldiServer. The user-service Deployment needs `user-internal-token`, so run it before ArgoCD syncs that change, or the new Pod waits until the Secret exists:

```bash
DB_PASSWORD=$(openssl rand -hex 24)
MQ_PASSWORD=$(openssl rand -hex 24)
TOKEN=$(openssl rand -hex 32)
kubectl exec -i -n postgresql postgresql-0 -- psql -U postgres -v ON_ERROR_STOP=1 -v password="$DB_PASSWORD" <<'SQL'
CREATE ROLE notification LOGIN PASSWORD :'password';
CREATE DATABASE notification OWNER notification;
SQL
kubectl exec -n rabbitmq rabbitmq-0 -- rabbitmqctl add_user notification "$MQ_PASSWORD"
kubectl exec -n rabbitmq rabbitmq-0 -- rabbitmqctl set_permissions -p shop notification '^(notification-service\..*|order\.events)$' '^notification-service\..*$' '^(notification-service\..*|order\.events)$'
kubectl create secret generic notification-database -n notification \
  --from-literal=url="postgres://notification:$DB_PASSWORD@postgresql.postgresql.svc:5432/notification?sslmode=disable"
kubectl create secret generic notification-rabbitmq -n notification \
  --from-literal=url="amqp://notification:$MQ_PASSWORD@rabbitmq.rabbitmq.svc:5672/shop"
kubectl create secret generic notification-user-token -n notification --from-literal=token="$TOKEN"
kubectl create secret generic user-internal-token -n user --from-literal=token="$TOKEN"
unset DB_PASSWORD MQ_PASSWORD TOKEN
```

### Add the tokens for the admin pages

Run this once, on AldiServer, before ArgoCD syncs the admin services. admin-service asks user-service with the token notification-service already uses, and order-service with a new one. The order-service Deployment needs `order-internal-token`, so until it exists the new order-service Pod waits and the old one keeps serving:

```bash
for NAMESPACE in admin web-admin; do
  kubectl create namespace "$NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -
done
USER_TOKEN=$(kubectl get secret user-internal-token -n user -o jsonpath='{.data.token}' | base64 -d)
ORDER_TOKEN=$(openssl rand -hex 32)
kubectl create secret generic admin-user-token -n admin --from-literal=token="$USER_TOKEN"
kubectl create secret generic admin-order-token -n admin --from-literal=token="$ORDER_TOKEN"
kubectl create secret generic order-internal-token -n order --from-literal=token="$ORDER_TOKEN"
unset USER_TOKEN ORDER_TOKEN
```

Then make an account an admin, as in [Make a user an admin](#make-a-user-an-admin), and log in at `/admin`.

For the *Produk* page, admin-service also needs catalog-service's token, and catalog-service refuses every change without it. Run this before ArgoCD syncs that change. Until both Secrets exist, the new catalog-service and admin-service Pods wait and the old ones keep serving:

```bash
CATALOG_TOKEN=$(openssl rand -hex 32)
kubectl create secret generic admin-catalog-token -n admin --from-literal=token="$CATALOG_TOKEN"
kubectl create secret generic catalog-internal-token -n catalog --from-literal=token="$CATALOG_TOKEN"
unset CATALOG_TOKEN
```

To change stock by hand, for example after a test order, send the change with that token:

```bash
TOKEN=$(kubectl get secret catalog-internal-token -n catalog -o jsonpath='{.data.token}' | base64 -d)
kubectl run stock-fix -n catalog --rm -i --restart=Never --image=curlimages/curl -- \
  -s -X PATCH -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"delta":1}' http://catalog-service.catalog.svc:8080/variants/1/stock
unset TOKEN
```

For the *Pengguna* page, user-service gets a second token that lists accounts and changes roles, and only admin-service has it, so the token notification-service uses can't make anyone an admin. Run this before ArgoCD syncs that change. Until both Secrets exist, the new user-service and admin-service Pods wait and the old ones keep serving:

```bash
USER_ADMIN_TOKEN=$(openssl rand -hex 32)
kubectl create secret generic admin-user-admin-token -n admin --from-literal=token="$USER_ADMIN_TOKEN"
kubectl create secret generic user-admin-token -n user --from-literal=token="$USER_ADMIN_TOKEN"
unset USER_ADMIN_TOKEN
```

### Photos

The shop's photos live in `/var/lib/aldi/photos` on AldiServer, outside any image. web reads them through the `web-photos` volume, read-only, and admin-service writes them through `admin-photos`, a second volume on the same folder, when an admin uploads a photo. admin-service runs as uid 1000, which owns `categories/` and `products/`, so nothing needs `sudo`. If those folders are ever made again by another user, give them back with `sudo chown -R 1000:1000 /var/lib/aldi/photos/categories /var/lib/aldi/photos/products`. SELinux is permissive on AldiServer; if it's ever made enforcing, the folder needs the `container_file_t` label for admin-service to write there.

## Open the shop to the internet

The Ingress in `web/` routes the domain, but nothing reaches it until Cloudflare knows about it. In the Cloudflare dashboard, add a Public Hostname to the tunnel: `prastika.aldiab.ai.id`, type `HTTP`, service `traefik.traefik.svc.cluster.local:80`.

## Monitoring

| Part | Does |
|---|---|
| Prometheus | Reads every metric each 30 seconds and keeps them for 30 days or 15 GB, whichever comes first, in `/var/lib/aldi/prometheus`. It has no login, so it has no Ingress |
| node-exporter | AldiServer's CPU, memory, disks and network. It shares the server's network, so it listens on port 9100 of the server |
| kube-state-metrics | Which Pods are ready, restarts and why a container last stopped, and Deployments missing replicas |
| kubelet and cAdvisor | Each container's CPU and memory, read through the kubelet with Prometheus's ServiceAccount |
| The shop's services | Requests, errors, latency and Go's memory, from each service's `/metrics` on port 9091, found through their Pods' `prometheus.io` annotations |
| RabbitMQ | Each queue's waiting and unacknowledged messages and its consumers, from the plugin on port 15692 |
| nats-exporter | NATS connections and messages, from NATS's monitoring port 8222 |
| postgres-exporter | Connections, commits and size of each database, logged in as the `monitoring` role |
| Alertmanager | Sends the alerts Prometheus raises to Telegram, grouped, and again every 4 hours (24 for warnings) while they last. Its silences are forgotten when it restarts |
| Grafana | The dashboards at `https://grafana.aldiab.ai.id`, with only an `admin` login. Its users and settings are in `/var/lib/aldi/grafana` |

The dashboards **Server**, **Kubernetes** and **Toko**, in the AldiServer folder, come from [`monitoring/dashboards/`](monitoring/dashboards/) and can't be changed in Grafana's pages. To change one, edit its JSON in a PR. A dashboard made in Grafana is kept in its database instead.

A Pod with the annotations `prometheus.io/scrape: "true"` and `prometheus.io/port` is read automatically, at `prometheus.io/path` or `/metrics`.

To query Prometheus directly, run this on AldiServer and browse to `http://localhost:9090` through an SSH tunnel:

```bash
kubectl port-forward -n monitoring svc/prometheus 9090:9090
```

### Alerts

The rules are in [`monitoring/alert-rules.yml`](monitoring/alert-rules.yml), each with a `critical` or `warning` severity and its Telegram text:

| Alert | When |
|---|---|
| TargetDown | A metrics endpoint can't be read for 5 minutes |
| DeploymentMissingReplicas, StatefulSetMissingReplicas | Fewer ready Pods than wanted for 10 minutes |
| PodCrashLooping | A container stuck in CrashLoopBackOff or failing to pull its image for 5 minutes |
| PodRestarting | More than 3 restarts in 30 minutes |
| HighErrorRate | Over 5% of a service's requests answer 5xx for 10 minutes, with some traffic |
| SlowResponses | A service's p95 latency is over 2 seconds for 15 minutes |
| PostgreSQLDown, PostgreSQLTooManyConnections | PostgreSQL unreachable for 2 minutes, or over 80% of its connections in use |
| NATSDown | No NATS numbers for 5 minutes |
| RabbitMQQueueBacklog, RabbitMQQueueWithoutConsumer | Over 100 messages waiting for 10 minutes, or messages with no consumer for 5 |
| DiskAlmostFull, DiskFull | Under 15% free for 10 minutes, or under 5% for 5 |
| MemoryLow, CPUBusy | Under 10% memory free for 10 minutes, or CPU over 90% for 30 |
| AlertsNotSent | Alertmanager failed to deliver a notification |

Firing alerts are listed in Prometheus at `/alerts`, and in Alertmanager, which also takes silences:

```bash
kubectl port-forward -n monitoring svc/alertmanager 9093:9093
```

### Connect Telegram

Run this once, on AldiServer, before ArgoCD syncs Alertmanager, or its Pod waits until the Secret exists. First make a bot by messaging `/newbot` to @BotFather in Telegram. Then open the new bot by its exact username, tap **Start** and send it any message, so it can find your chat. Run this in your own SSH session, so the token isn't recorded anywhere else:

```bash
read -rsp "Bot token: " TOKEN; echo
curl -s "https://api.telegram.org/bot$TOKEN/getMe" | jq -r .result.username
CHAT_ID=$(curl -s "https://api.telegram.org/bot$TOKEN/getUpdates" | jq -r '[.. | objects | .chat? | objects | .id] | last // empty')
echo "Chat ID: $CHAT_ID"
[ -n "$CHAT_ID" ] && kubectl create secret generic alertmanager-telegram -n monitoring \
  --from-literal=token="$TOKEN" --from-literal=chat_id="$CHAT_ID"
unset TOKEN CHAT_ID
```

The first line printed is the bot's username, which shows the token is right. If the chat ID comes out empty, the bot has had no message yet: send it one and run this again. To change either later, delete the Secret, create it again, and restart Alertmanager with `kubectl rollout restart deploy/alertmanager -n monitoring`.

### Create the Grafana login

Run this once, on AldiServer, before the first sync. Grafana only reads it when it creates its database, so later changes to the password are made in Grafana:

```bash
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
GRAFANA_PASSWORD=$(openssl rand -base64 18)
kubectl create secret generic grafana-admin -n monitoring --from-literal=password="$GRAFANA_PASSWORD"
echo "Grafana admin password: $GRAFANA_PASSWORD"
unset GRAFANA_PASSWORD
```

If the password is lost, reset it with `kubectl exec -n monitoring deploy/grafana -- grafana cli admin reset-admin-password '<new password>'`.

### Add the PostgreSQL login for postgres-exporter

Run this once, on AldiServer, before ArgoCD syncs postgres-exporter, or its Pod waits until the Secret exists. `pg_monitor` can read PostgreSQL's statistics, but no table:

```bash
DB_PASSWORD=$(openssl rand -hex 24)
kubectl exec -i -n postgresql postgresql-0 -- psql -U postgres -v ON_ERROR_STOP=1 -v password="$DB_PASSWORD" <<'SQL'
CREATE ROLE monitoring LOGIN PASSWORD :'password';
GRANT pg_monitor TO monitoring;
SQL
kubectl create secret generic postgres-exporter -n monitoring \
  --from-literal=url="postgres://monitoring:$DB_PASSWORD@postgresql.postgresql.svc:5432/postgres?sslmode=disable"
unset DB_PASSWORD
```

### Open Grafana to the internet

In the Cloudflare dashboard, add a Public Hostname to the tunnel: `grafana.aldiab.ai.id`, type `HTTP`, service `traefik.traefik.svc.cluster.local:80`, the same as the shop's.

## PostgreSQL

One instance serves every application, each with its own database and role. The data is in `/var/lib/aldi/postgresql` on AldiServer and has no backup yet.

Create the superuser Secret, once, before the first sync:

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
kubectl create secret generic catalog-database -n catalog \
  --from-literal=url="postgres://catalog:$PASSWORD@postgresql.postgresql.svc:5432/catalog?sslmode=disable"
```

### Add a database for cart-service

Run this once, on AldiServer, before ArgoCD syncs cart-service, or its Pod waits until the Secret exists:

```bash
kubectl create namespace cart --dry-run=client -o yaml | kubectl apply -f -
DB_PASSWORD=$(openssl rand -hex 24)
kubectl exec -i -n postgresql postgresql-0 -- psql -U postgres -v ON_ERROR_STOP=1 -v password="$DB_PASSWORD" <<'SQL'
CREATE ROLE cart LOGIN PASSWORD :'password';
CREATE DATABASE cart OWNER cart;
SQL
kubectl create secret generic cart-database -n cart \
  --from-literal=url="postgres://cart:$DB_PASSWORD@postgresql.postgresql.svc:5432/cart?sslmode=disable"
unset DB_PASSWORD
```

### Add a database and key for user-service

Run this once, on AldiServer, after the `postgresql-0` Pod is ready. The key signs login tokens. A new key logs everyone out, because tokens signed with the old one stop working:

```bash
PASSWORD=$(openssl rand -hex 24)
kubectl exec -i -n postgresql postgresql-0 -- psql -U postgres -v ON_ERROR_STOP=1 -v password="$PASSWORD" <<'SQL'
CREATE ROLE user_service LOGIN PASSWORD :'password';
CREATE DATABASE users OWNER user_service;
SQL
kubectl create secret generic user-database -n user \
  --from-literal=url="postgres://user_service:$PASSWORD@postgresql.postgresql.svc:5432/users?sslmode=disable"
unset PASSWORD
openssl genpkey -algorithm ed25519 | kubectl create secret generic user-service-jwt -n user --from-file=private-key=/dev/stdin
```

### Make a user an admin

New accounts are customers. An admin can make another account an admin, or take the role away, on the *Pengguna* page at `/admin/#pengguna`. The first admin, or one locked out, is made by hand on AldiServer. admin-service checks the role with user-service on every request, so a change applies at once, also for taking the role away:

```bash
kubectl exec -n postgresql postgresql-0 -- psql -U postgres -d users -c "UPDATE users SET role = 'admin', updated_at = now() WHERE email = 'name@example.com'"
```

## RabbitMQ

One broker carries the events between applications. The data is in `/var/lib/aldi/rabbitmq` on AldiServer. It is only reachable inside the cluster, at `rabbitmq.rabbitmq.svc:5672`.

Create the admin Secret, once, before the first sync. RabbitMQ only reads this password when its data folder is empty, so changing the Secret later doesn't change the password:

```bash
kubectl create secret generic rabbitmq-admin -n rabbitmq \
  --from-literal=password="$(openssl rand -hex 24)"
```

### Add a user for catalog-service

Applications share the `shop` vhost, each with its own user. catalog-service may only declare and publish to its `catalog.events` exchange. Run this once, on AldiServer, before the Deployment that needs the Secret:

```bash
PASSWORD=$(openssl rand -hex 24)
kubectl exec -n rabbitmq rabbitmq-0 -- rabbitmqctl add_vhost shop
kubectl exec -n rabbitmq rabbitmq-0 -- rabbitmqctl add_user catalog "$PASSWORD"
kubectl exec -n rabbitmq rabbitmq-0 -- rabbitmqctl set_permissions -p shop catalog '^catalog\.events$' '^catalog\.events$' '^$'
kubectl create secret generic catalog-rabbitmq -n catalog \
  --from-literal=url="amqp://catalog:$PASSWORD@rabbitmq.rabbitmq.svc:5672/shop"
unset PASSWORD
```

### Add the databases and users for the order saga

Run this once, on AldiServer. It also lets catalog-service read order events, so run it before deploying a catalog-service that reserves stock:

```bash
kubectl exec -n rabbitmq rabbitmq-0 -- rabbitmqctl set_permissions -p shop catalog '^(catalog-service\..*|catalog\.events|order\.events)$' '^(catalog-service\..*|catalog\.events)$' '^(catalog-service\..*|order\.events)$'
for APP in order payment; do
  DB_PASSWORD=$(openssl rand -hex 24)
  MQ_PASSWORD=$(openssl rand -hex 24)
  kubectl exec -i -n postgresql postgresql-0 -- psql -U postgres -v ON_ERROR_STOP=1 -v password="$DB_PASSWORD" -v app="$APP" <<'SQL'
CREATE ROLE :"app" LOGIN PASSWORD :'password';
CREATE DATABASE :"app" OWNER :"app";
SQL
  kubectl exec -n rabbitmq rabbitmq-0 -- rabbitmqctl add_user "$APP" "$MQ_PASSWORD"
  kubectl create secret generic "$APP-database" -n "$APP" \
    --from-literal=url="postgres://$APP:$DB_PASSWORD@postgresql.postgresql.svc:5432/$APP?sslmode=disable"
  kubectl create secret generic "$APP-rabbitmq" -n "$APP" \
    --from-literal=url="amqp://$APP:$MQ_PASSWORD@rabbitmq.rabbitmq.svc:5672/shop"
done
unset DB_PASSWORD MQ_PASSWORD
kubectl exec -n rabbitmq rabbitmq-0 -- rabbitmqctl set_permissions -p shop order '^(order-service\..*|order\.events|catalog\.events|payment\.events)$' '^(order-service\..*|order\.events)$' '^(order-service\..*|catalog\.events|payment\.events)$'
kubectl exec -n rabbitmq rabbitmq-0 -- rabbitmqctl set_permissions -p shop payment '^(payment-service\..*|order\.events|payment\.events)$' '^(payment-service\..*|payment\.events)$' '^(payment-service\..*|order\.events)$'
```

### Open the management UI

To open the management UI, run this on AldiServer, then browse to `http://localhost:15672` through an SSH tunnel and log in as `admin`:

```bash
kubectl port-forward -n rabbitmq svc/rabbitmq 15672:15672
kubectl get secret rabbitmq-admin -n rabbitmq -o jsonpath='{.data.password}' | base64 -d; echo
```

## Elasticsearch

One node holds the search index. The data is in `/var/lib/aldi/elasticsearch` on AldiServer, and it can be rebuilt from the catalog. It is only reachable inside the cluster, at `elasticsearch.elasticsearch.svc:9200`.

Create the admin Secret, once, before the first sync. Unlike RabbitMQ, a changed password takes effect when the Pod restarts:

```bash
kubectl create secret generic elasticsearch-admin -n elasticsearch \
  --from-literal=password="$(openssl rand -hex 24)"
```

### Add the users for search-service

search-service may only use its `products` index in Elasticsearch, and its own queue in RabbitMQ. Run this once, on AldiServer, after `elasticsearch-0` and `rabbitmq-0` are ready and the `shop` vhost exists:

```bash
ADMIN=$(kubectl get secret elasticsearch-admin -n elasticsearch -o jsonpath='{.data.password}' | base64 -d)
ES_PASSWORD=$(openssl rand -hex 24)
MQ_PASSWORD=$(openssl rand -hex 24)
kubectl exec -n elasticsearch elasticsearch-0 -- curl -sS -u "elastic:$ADMIN" -XPUT localhost:9200/_security/role/search_service \
  -H 'Content-Type: application/json' -d '{"cluster":["monitor"],"indices":[{"names":["products"],"privileges":["all"]}]}'
kubectl exec -n elasticsearch elasticsearch-0 -- curl -sS -u "elastic:$ADMIN" -XPUT localhost:9200/_security/user/search_service \
  -H 'Content-Type: application/json' -d "{\"password\":\"$ES_PASSWORD\",\"roles\":[\"search_service\"]}"
kubectl exec -n rabbitmq rabbitmq-0 -- rabbitmqctl add_user search "$MQ_PASSWORD"
kubectl exec -n rabbitmq rabbitmq-0 -- rabbitmqctl set_permissions -p shop search '^(search-service\..*|catalog\.events)$' '^search-service\..*$' '^(search-service\..*|catalog\.events)$'
kubectl create secret generic search-service -n search \
  --from-literal=elasticsearch-url="http://search_service:$ES_PASSWORD@elasticsearch.elasticsearch.svc:9200" \
  --from-literal=rabbitmq-url="amqp://search:$MQ_PASSWORD@rabbitmq.rabbitmq.svc:5672/shop"
unset ADMIN ES_PASSWORD MQ_PASSWORD
```
