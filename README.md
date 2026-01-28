# Caddy (xcaddy custom) + CrowdSec (LAPI + AppSec) – Docker stack

This setup builds a custom Caddy binary with the following xcaddy modules:
- github.com/sjtug/caddy2-filter
- github.com/caddy-dns/cloudflare
- github.com/hslatman/caddy-crowdsec-bouncer/http
- github.com/hslatman/caddy-crowdsec-bouncer/appsec

It also runs CrowdSec in its own container (LAPI + AppSec), with collections installed at startup.

## Layout
- build/Dockerfile.caddy        -> builds the custom Caddy image
- deploy/compose.yaml           -> runs the full stack
- deploy/Caddyfile              -> example config (MUST be changed to your domain)
- deploy/crowdsec/acquis.d/*    -> acquis for Caddy + AppSec
- deploy/dotenv_example           -> copy to deploy/.env

## 1) Prepare
Create the bind-mount directories on the host (Caddy runs as UID/GID 1000 in the image):

```bash
sudo mkdir -p /opt/caddy/{data,config,logs} /opt/crowdsec/{data,config}
sudo chown -R 1000:1000 /opt/caddy/{data,config,logs}
sudo chown -R 0:0 /opt/crowdsec/{data,config}
```

Go to the deploy folder and create `.env`:

```bash
cd deploy
cp dotenv_example .env
nano .env
```

Update `your.domain.tld` in `deploy/Caddyfile` to your actual domain.

## 2) Start the stack (builds the Caddy image on first run)
```bash
docker compose up -d --build
```

## 3) Generate a CrowdSec bouncer key (one-time)
Run:

```bash
docker compose exec crowdsec cscli bouncers add caddyDmz
```

Copy the key that is printed and set it in `deploy/.env` as `CROWDSEC_API_TOKEN=...`.

Restart Caddy:

```bash
docker compose restart caddy
```

## 4) Check status
```bash
docker compose logs -f caddy
docker compose exec crowdsec cscli collections list
docker compose exec crowdsec cscli metrics
```

## Notes
- CrowdSec reads Caddy logs from `/opt/caddy/logs` via a shared bind mount.
- AppSec must listen on 0.0.0.0:7422 in the container so Caddy can reach it.
