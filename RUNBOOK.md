# Runbook — add a new blue-green app on datahorde

How to stand up a new Django app behind HTTPS on **datahorde**
(`roots.origamisoftware.com`, 47.14.230.188) using this repo's shared
blue-green deploy workflow. Written from the melee bring-up; orge / tarmar /
vct / jobmaster / pa are all deployed this same way — copy the closest one.

Every app gets: two systemd units (blue + green) on a dedicated port pair, an
nginx upstream that points at whichever colour is live, an nginx server block
with its own TLS cert, and a GitHub Actions `deploy.yml` that calls
`smarks/ops-workflows/.github/workflows/deploy-blue-green.yml` to SSH in and run
the app's `deploy.sh` on every push to `main`.

---

## 0. Pick a free port pair FIRST

Ports are the #1 footgun — melee originally shipped 9070/9071, which **already
belonged to virtual_card_table**, so its units crash-looped silently. Always
check before choosing:

```bash
grep -REn 'bind 127.0.0.1:[0-9]+' /etc/systemd/system/*.service   # declared
sudo ss -ltnp 'sport >= :9060 and sport <= :9099'                 # actually listening
```

Current allocation (blue / green):

| app                 | blue | green |
|---------------------|------|-------|
| tarmar              | 9066 | 9067  |
| orge                | 9068 | 9069  |
| virtual_card_table  | 9070 | 9071  |
| melee               | 9072 | 9073  |
| jobmaster           | 9080 | 9081  |
| pa                  | 9090 | 9091  |

**Next free pair: 9074 / 9075.** Pick the next free consecutive pair and use it
consistently everywhere below.

---

## 1. In the new app's repo

Copy these from melee (or the most similar app) and swap names/ports:

1. **`deploy.sh`** — set `cd <app-dir>`, `BLUE_PORT` / `GREEN_PORT` to your pair,
   `UPSTREAM_CONF=/etc/nginx/conf.d/<app>-upstream.conf`, `STATE_FILE=<app-dir>/.deploy-state`.
   The health check must send the **real Host header**, not bare `127.0.0.1`:
   ```bash
   curl -sf -H "Host: <app>.origamisoftware.com" "http://127.0.0.1:$port/"
   ```
   (Django's `ALLOWED_HOSTS` is the public hostname only, so a `127.0.0.1`
   request returns 400 and the check would never pass. A 3xx HTTPS-redirect
   still counts as healthy.)
2. **`<app>-blue.service` / `<app>-green.service`** — set `Description`,
   `WorkingDirectory`, `EnvironmentFile=<app-dir>/.env`, and the gunicorn
   `--bind 127.0.0.1:<blue|green port>` + `<project>.wsgi:application`.
   Keep `--workers 1` only if the app holds in-memory state (melee does);
   otherwise raise it.
3. **`.github/workflows/deploy.yml`** — copy verbatim, change only `app-dir`:
   ```yaml
   jobs:
     deploy:
       uses: smarks/ops-workflows/.github/workflows/deploy-blue-green.yml@main
       with:
         app-dir: /home/sam/dev/<app>
         command: ${{ inputs.command || '' }}
       secrets:
         ssh-key: ${{ secrets.DEPLOY_SSH_KEY }}
   ```
   The reusable workflow defaults `host: roots.origamisoftware.com` (= datahorde)
   and `user: sam`; only override if deploying elsewhere.
4. **Repo secrets:** `DEPLOY_SSH_KEY` (private key authorized for `sam@datahorde`)
   and `MAIL_PASSWORD` (for the failure-notify job).

### ⚠️ Repo visibility gotcha
A **public** app repo **cannot** call a reusable workflow stored in a **private**
repo — the run fails at validation (0 s, no jobs, "workflow file issue"), and it
fails *silently* on every push. This repo (`ops-workflows`) was made **public**
precisely so the public apps (e.g. melee) can use it. So:
- app repo **private**  → fine either way.
- app repo **public**   → `ops-workflows` must stay **public** (it is). If you
  ever take it private, every public app's auto-deploy breaks.

---

## 2. On datahorde (server)

```bash
# clone + branch
git clone https://github.com/smarks/<app>.git /home/sam/dev/<app>
cd /home/sam/dev/<app>            # stay on main — deploy.sh git-pulls the current branch

# .env (NOT committed — it's gitignored; holds the secret)
#   DJANGO_SECRET_KEY=...
#   DJANGO_DEBUG=0
#   DJANGO_ALLOWED_HOSTS=<app>.origamisoftware.com
# venv
python3 -m venv .venv && .venv/bin/pip install -U pip && .venv/bin/pip install -r requirements.txt

# systemd units (idempotent)
sudo cp <app>-blue.service <app>-green.service /etc/systemd/system/
sudo systemctl daemon-reload && sudo systemctl enable --now <app>-blue <app>-green
# verify BOTH actually serve (use the Host header — a bare 127.0.0.1 returns 400):
curl -sf -H "Host: <app>.origamisoftware.com" http://127.0.0.1:<blue>/ && echo "blue OK"
curl -sf -H "Host: <app>.origamisoftware.com" http://127.0.0.1:<green>/ && echo "green OK"
# if a unit crash-loops on "Address already in use" -> your ports aren't free (back to step 0)

# nginx upstream -> live (blue) port  + seed deploy state
echo 'upstream <app>_backend { server 127.0.0.1:<blue>; }' | sudo tee /etc/nginx/conf.d/<app>-upstream.conf
echo blue | tee /home/sam/dev/<app>/.deploy-state

# nginx server block (HTTP bootstrap; certbot adds 443 + redirect). Mirror orge:
sudo tee /etc/nginx/sites-available/<app> > /dev/null << 'NGINX'
server {
    listen 80;
    server_name <app>.origamisoftware.com;
    location / {
        proxy_pass http://<app>_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
NGINX
sudo ln -sfn /etc/nginx/sites-available/<app> /etc/nginx/sites-enabled/<app>
sudo nginx -t && sudo systemctl reload nginx

# TLS — give each app its OWN cert (safer than --expand-ing the shared
# origamisoftware.com cert, which serves 6 live domains):
sudo certbot --nginx -d <app>.origamisoftware.com --non-interactive --redirect

# verify
curl -sI https://<app>.origamisoftware.com | head -1     # expect HTTP/1.1 200 OK
```

Make sure DNS for `<app>.origamisoftware.com` resolves to **47.14.230.188**
(CNAME to `roots.origamisoftware.com` / `tarmar.duckdns.org`) before running
certbot — the HTTP-01 challenge needs it reachable on port 80.

---

## 3. Done / steady state

- `main` is the deploy branch — keep the datahorde checkout **on `main`**.
  `deploy.sh` self-updates with `git pull` but never `git checkout`s, so a checkout
  left on a feature branch will never pick up new `main` commits.
- Every push to the app's `main` now auto-deploys: GitHub → SSH → `deploy.sh` on
  datahorde → tests → migrate/collectstatic → restart the **inactive** colour →
  health-check → flip the nginx upstream. Old colour stays up for
  `./deploy.sh rollback`.
- `./deploy.sh status` shows the live colour; `./deploy.sh --force` redeploys even
  with no new commits.

## Gotchas learned (melee, 2026-06-29)
1. **Verify free ports first** — silent crash-loops otherwise (9070/9071 were vct's).
2. **Host header in the health check** — `ALLOWED_HOSTS` is the public host only.
3. **Public app repo ⇒ ops-workflows must be public.**
4. **Own cert per app**, don't `--expand` the shared one.
5. A green `curl 127.0.0.1:<port>` can be a *neighbouring* app answering — confirm
   it's actually yours (check the systemd unit / the response).
