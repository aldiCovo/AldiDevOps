# saham

A personal LQ45 market brief, kept apart from the shop in its own ArgoCD Application, namespace and PostgreSQL. Nothing here is shared or sold.

| Folder | What |
| --- | --- |
| [`postgresql/`](postgresql/) | PostgreSQL 18 for the paper trades, momentum months and sent briefs. Data on AldiServer at `/var/lib/aldi/saham-postgresql`. |
| [`signal-service/`](signal-service/) | Reads LQ45 and IHSG prices from Yahoo after each trading day (17:00 WIB), paper-trades every rule against random entries, and sends the brief to Telegram. Code in `aldiCovo/AldiApps`, `saham/signal-service`. |

The rules are a screener, not advice: none of them beat costs in the backtest. The paper trading is there to see whether any does from now on.

## First setup

The namespace and the Secrets are made by hand, before the Application, so no password or bot token enters this public repo.

```sh
kubectl create namespace saham

# The database password: made up once, never typed again.
kubectl -n saham create secret generic saham-postgresql \
  --from-literal=password="$(openssl rand -hex 24)"

# Optional: the bot that sends the brief, made with @BotFather, and the chat it writes to.
# Without it the brief is only written to the log and to /api/latest.
read -s TOKEN && read CHAT && kubectl -n saham create secret generic saham-telegram \
  --from-literal=token="$TOKEN" --from-literal=chat_id="$CHAT"

kubectl apply -f k8s/argocd/saham-application.yaml
```

The image `ghcr.io/aldicovo/signal-service` must be public on GitHub, as the shop's are.

## Day to day

```sh
# The last brief.
kubectl -n saham port-forward deploy/signal-service 8080 &
curl -s localhost:8080/api/latest

# Run now. force=1 also runs on a day without trading, or one already done.
curl -X POST 'localhost:8080/run?force=1'
```

Prometheus reads the job's metrics. `SahamJobFailed` goes to Telegram when a run fails, and `SahamJobStale` when none has worked for 80 hours. The Grafana dashboards **Kubernetes** and **Log** show the namespace under the group **Saham**.
