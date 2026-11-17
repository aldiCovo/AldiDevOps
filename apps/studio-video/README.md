# studio

**Studio Video** at **https://studio.aldiab.ai.id**: an idea becomes an AI storyboard, Edge TTS narration and AI-written Remotion code, which is type-checked here and rendered by a worker on the Windows laptop. Code in `aldiCovo/AldiApps`, `studio-video`.

| What | Where |
| --- | --- |
| Videos, scripts, code, Claude sessions | `/var/lib/aldi/studio-video/jobs` on AldiServer |
| Narration the worker downloads | `/var/lib/aldi/studio-video/public-jobs` |
| Optional background music (`.mp3`, `.m4a`, `.wav`) | `/var/lib/aldi/studio-video/music` |
| Claude Code's home, for `--resume` on revisions | `/var/lib/aldi/studio-video/home` |
| `.env` with upload keys and the tokens the app rotates | `/var/lib/aldi/studio-video/config/.env` |

The queue lives in the process's memory, so the Deployment has one replica and `Recreate`. A restart loses a render in progress (start it again with **Render ulang**), never the files.

## First setup

The namespace and the Secrets are made by hand, before the Application, so no token enters this public repo.

```sh
kubectl create namespace studio

# Shared with the render laptop. Made up once; read it back below when setting up the laptop.
kubectl -n studio create secret generic studio-worker \
  --from-literal=token="$(openssl rand -hex 32)"

# Claude Pro: run `claude setup-token` on a computer logged in to Claude, then paste the token here.
read -s T && kubectl -n studio create secret generic studio-claude --from-literal=token="$T" && unset T

kubectl apply -f k8s/argocd/studio-application.yaml
```

In the Cloudflare dashboard:

1. **Networks → Tunnels →** the tunnel **→ Public Hostname**: `studio.aldiab.ai.id`, type `HTTP`, service `traefik.traefik.svc.cluster.local:80`, the same as the shop's.
2. **Zero Trust → Access → Applications**, two self-hosted applications:
   - `studio.aldiab.ai.id`, policy **Allow**, Include **Emails**: your own address. You log in with a code sent to that email.
   - `studio.aldiab.ai.id` with path `api/worker`, policy **Bypass**, Include **Everyone**. The more specific path wins, so the worker reaches the app, which checks its `Authorization: Bearer` token itself.

## The render laptop

```sh
# The value for WORKER_TOKEN in the laptop's .env.
kubectl -n studio get secret studio-worker -o jsonpath='{.data.token}' | base64 -d; echo
```

The laptop's `.env`:

```
STUDIO_URL=https://studio.aldiab.ai.id
WORKER_TOKEN=<the value above>
WORKER_NAME=laptop-windows
```

## Upload keys

YouTube, TikTok, Instagram and Facebook keys go in `.env` (see `.env.example` in the code), not in the Deployment, because the app rewrites `.env` when TikTok rotates its refresh token. The app reads `.env` only when it starts.

```sh
sudo vi /var/lib/aldi/studio-video/config/.env
kubectl -n studio rollout restart deployment studio
```

## Limits

Cloudflare cuts a request that sends nothing for 100 seconds. `/api/prompt` and `/api/plan` wait for Claude in one request, so the app sends a space every 15 seconds until the answer is ready. Uploads from the worker are 8 MB pieces, under Cloudflare's 100 MB limit.
