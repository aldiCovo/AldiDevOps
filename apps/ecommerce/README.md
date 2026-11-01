# Online shop

Everything for the shop, with one folder per service and per tool it uses.

| Folder | What it is | Applied with |
|---|---|---|
| [`catalog-service/`](catalog-service/) | Products and stock | ArgoCD |
| [`search-service/`](search-service/) | Product search | ArgoCD |
| [`user-service/`](user-service/) | Accounts and login | ArgoCD |
| [`postgresql/`](postgresql/) | Database | `kubectl apply -f apps/ecommerce/postgresql/namespace.yaml`, create the `postgresql-superuser` Secret as below, then `kubectl apply -f apps/ecommerce/postgresql/` |
| [`rabbitmq/`](rabbitmq/) | Events between services | `kubectl apply -f apps/ecommerce/rabbitmq/namespace.yaml`, create the `rabbitmq-admin` Secret as below, then `kubectl apply -f apps/ecommerce/rabbitmq/` |
| [`elasticsearch/`](elasticsearch/) | Search index | `kubectl apply -f apps/ecommerce/elasticsearch/namespace.yaml`, create the `elasticsearch-admin` Secret as below, then `kubectl apply -f apps/ecommerce/elasticsearch/` |

The ArgoCD Applications are in [`k8s/argocd/`](../../k8s/argocd/). The other folders are applied by hand.

## PostgreSQL

One instance serves every application, each with its own database and role. The data is in `/var/lib/aldi/postgresql` on AldiServer and has no backup yet.

Create the superuser Secret, once, before the first `kubectl apply -f apps/ecommerce/postgresql/`:

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

### Add a database and key for user-service

Run this once, on AldiServer, after the `postgresql-0` Pod is ready. The key signs login tokens. A new key logs everyone out, because tokens signed with the old one stop working:

```bash
PASSWORD=$(openssl rand -hex 24)
kubectl exec -i -n postgresql postgresql-0 -- psql -U postgres -v ON_ERROR_STOP=1 -v password="$PASSWORD" <<'SQL'
CREATE ROLE user_service LOGIN PASSWORD :'password';
CREATE DATABASE users OWNER user_service;
SQL
kubectl create namespace user
kubectl create secret generic user-database -n user \
  --from-literal=url="postgres://user_service:$PASSWORD@postgresql.postgresql.svc:5432/users?sslmode=disable"
unset PASSWORD
openssl genpkey -algorithm ed25519 | kubectl create secret generic user-service-jwt -n user --from-file=private-key=/dev/stdin
```

### Make a user an admin

New accounts are customers, and the API can't change that. Run this on AldiServer. The new role reaches tokens at the user's next login:

```bash
kubectl exec -n postgresql postgresql-0 -- psql -U postgres -d users -c "UPDATE users SET role = 'admin', updated_at = now() WHERE email = 'name@example.com'"
```

## RabbitMQ

One broker carries the events between applications. The data is in `/var/lib/aldi/rabbitmq` on AldiServer. It is only reachable inside the cluster, at `rabbitmq.rabbitmq.svc:5672`.

Create the admin Secret, once, before the first `kubectl apply -f apps/ecommerce/rabbitmq/`. RabbitMQ only reads this password when its data folder is empty, so changing the Secret later doesn't change the password:

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

### Open the management UI

To open the management UI, run this on AldiServer, then browse to `http://localhost:15672` through an SSH tunnel and log in as `admin`:

```bash
kubectl port-forward -n rabbitmq svc/rabbitmq 15672:15672
kubectl get secret rabbitmq-admin -n rabbitmq -o jsonpath='{.data.password}' | base64 -d; echo
```

## Elasticsearch

One node holds the search index. The data is in `/var/lib/aldi/elasticsearch` on AldiServer, and it can be rebuilt from the catalog. It is only reachable inside the cluster, at `elasticsearch.elasticsearch.svc:9200`.

Create the admin Secret, once, before the first `kubectl apply -f apps/ecommerce/elasticsearch/`. Unlike RabbitMQ, a changed password takes effect when the Pod restarts:

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
kubectl create namespace search
kubectl create secret generic search-service -n search \
  --from-literal=elasticsearch-url="http://search_service:$ES_PASSWORD@elasticsearch.elasticsearch.svc:9200" \
  --from-literal=rabbitmq-url="amqp://search:$MQ_PASSWORD@rabbitmq.rabbitmq.svc:5672/shop"
unset ADMIN ES_PASSWORD MQ_PASSWORD
```
