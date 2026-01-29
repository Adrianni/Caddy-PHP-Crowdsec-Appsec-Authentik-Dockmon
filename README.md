# Caddy (xcaddy custom) + CrowdSec (LAPI + AppSec) + Authentik – Docker stack

This setup builds a custom Caddy binary with the following xcaddy modules:
- github.com/sjtug/caddy2-filter
- github.com/caddy-dns/cloudflare
- github.com/hslatman/caddy-crowdsec-bouncer/http
- github.com/hslatman/caddy-crowdsec-bouncer/appsec

It also runs CrowdSec in its own container (LAPI + AppSec), with collections installed at startup, plus Authentik behind Caddy.

## Layout
- build/Dockerfile.caddy        -> builds the custom Caddy image
- build/Dockerfile.php          -> builds the custom PHP-FPM image
- deploy/compose.yaml           -> runs the full stack
- deploy/Caddyfile              -> localhost Caddy config with CrowdSec/AppSec + Authentik
- deploy/crowdsec/acquis.d/*    -> acquis for Caddy + AppSec
- deploy/dotenv_example         -> copy to deploy/.env

## 1) Prepare
Create the bind-mount directories on the host (Caddy runs as UID/GID 1000 in the image):

```bash
sudo mkdir -p /opt/caddy/{www-data,data,config,logs} /opt/crowdsec/{data,config} \
  /opt/crowdsec/data/acquis.d \
  /opt/Authentik/{postgres,redis,media,custom-templates} \
  /opt/php-fpm/config
sudo chown -R 1000:1000 /opt/caddy/{www-data,data,config,logs}
sudo chown -R 0:0 /opt/crowdsec/{data,config}
sudo chown -R 1000:1000 /opt/Authentik/{postgres,redis,media,custom-templates}
sudo chown -R 1000:1000 /opt/php-fpm/config
sudo find /opt/caddy/www-data -type d -exec chmod 755 {} \;
sudo find /opt/caddy/www-data -type f -exec chmod 644 {} \;
```

Copy the CrowdSec acquis configuration into the host data path (so both CrowdSec + AppSec config live under `/opt/crowdsec/data`):

```bash
sudo cp -a deploy/crowdsec/acquis.d/. /opt/crowdsec/data/acquis.d/
```

Go to the deploy folder and create `.env`:

```bash
cd deploy
cp dotenv_example .env
nano .env
```

Set `CROWDSEC_API_TOKEN`, `AUTHENTIK_SECRET_KEY`, and `AUTHENTIK_POSTGRES_PASSWORD` in `deploy/.env`.

## 2) Start the stack (builds the Caddy + PHP-FPM images on first run)
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

## 5) Access
- `http://localhost` shows the validation page.
- `http://localhost/authentik` proxies to Authentik.

## Notes
- CrowdSec reads Caddy logs from `/opt/caddy/logs` via a shared bind mount.
- AppSec must listen on 0.0.0.0:7422 in the container so Caddy can reach it.
- Authentik images are pinned to a specific version in `deploy/compose.yaml` for reproducible upgrades.

## Updating packages and images
When something needs updating, pull new images and rebuild the custom Caddy image:

```bash
cd deploy
docker compose pull
docker compose up -d --build
```

If you want newer Authentik, update the image tag in `deploy/compose.yaml` and re-run the steps above.

If you want newer PHP, update `PHP_VERSION` in `deploy/compose.yaml` and rebuild the stack.
