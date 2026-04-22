# LGTM Observability Stack - Laravel EC2 Agent Template (Docker Alloy)

This guide is the reusable template for each Laravel EC2 node.

- Monitoring hub setup is in [aws-setup-docker-monitoringhub.md](aws-setup-docker-monitoringhub.md).
- Laravel app stack (Nginx/PHP-FPM/PHP) stays native on host.
- Only monitoring components run in Docker (`alloy`, `nginx-exporter`, `phpfpm-exporter`).

## Table of Contents

1. [Node Variables (Template Input)](#1-node-variables-template-input)
2. [Prerequisites](#2-prerequisites)
3. [Enable Host Status Endpoints](#3-enable-host-status-endpoints)
4. [Deploy Agent Stack](#4-deploy-agent-stack)
5. [Laravel Cleanup](#5-laravel-cleanup)
6. [Validation Checklist](#6-validation-checklist)
7. [Operations](#7-operations)

## 1. Node Variables (Template Input)

For each Laravel EC2, define these values before deploy:

- `HUB_IP` = monitoring hub private IP (for `3100`, `9009`)
- `INSTANCE_NAME` = unique node label (example: `api-prod-01`)
- `ENVIRONMENT` = `production`/`staging`
- `TZ_NAME` = IANA timezone (example: `Asia/Kuala_Lumpur`)
- `LARAVEL_LOG_DIR` = Laravel logs directory on that node
- `LARAVEL_APP_DIR` = Laravel project root on that node

Forge note:
- Many Forge deployments use paths under `/home/forge/SITE_NAME/...`.
- Do not assume `/data/sso-api/...`; confirm each node path first.

## 2. Prerequisites

From [aws-setup-docker-monitoringhub.md](aws-setup-docker-monitoringhub.md), ensure:
- Hub is healthy (`docker compose up -d` completed on hub)
- Hub SG allows inbound from app SG on `3100`, `9009`
- App node has outbound access to hub

Install Docker on each Laravel EC2:

```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
sudo apt install -y docker-compose-plugin

# re-login
exit
```

Verify after login:

```bash
docker --version
docker compose version
docker run --rm hello-world
```

## 3. Enable Host Status Endpoints

### 3.1 Enable PHP-FPM status path

```bash
sudo sed -i 's#^;*pm.status_path = .*#pm.status_path = /fpm-status#' /etc/php/*/fpm/pool.d/www.conf
sudo systemctl restart php*-fpm
```

### 3.2 Add Nginx status server (`:8080`)

```bash
ls /run/php/

sudo tee /etc/nginx/conf.d/stub_status.conf > /dev/null << 'EONGINX'
server {
    listen 8080;
    server_name localhost;

    location = /nginx_status {
        stub_status on;
        allow 127.0.0.1;
        deny all;
    }

    location = /fpm-status {
        fastcgi_pass unix:/run/php/php-fpm.sock;    # adjust if needed
        fastcgi_param SCRIPT_NAME     /fpm-status;
        fastcgi_param SCRIPT_FILENAME /fpm-status.php;
        fastcgi_param REQUEST_URI     /fpm-status;
        fastcgi_param QUERY_STRING    $query_string;
        fastcgi_param REQUEST_METHOD  $request_method;
        allow 127.0.0.1;
        deny all;
    }
}
EONGINX

sudo nginx -t && sudo systemctl reload nginx
```

### 3.3 Verify endpoints

```bash
curl -s http://127.0.0.1:8080/nginx_status
curl -s http://127.0.0.1:8080/fpm-status
```

## 4. Deploy Agent Stack

### 4.1 Copy files to node

```bash
cd /opt
sudo mkdir -p monitoring && sudo chown $USER:$USER monitoring
git clone YOUR_REPOSITORY_URL monitoring
# example: git clone git@github.com:your-org/your-repo.git monitoring
# Alternative: if you copy only agent files:
# scp -r alloy-docker/ user@laravel-ec2:/opt/monitoring/

cd /opt/monitoring/alloy-docker
```

### 4.2 Update `config.alloy`

Edit:

```bash
nano /opt/monitoring/alloy-docker/config.alloy
```

Replace values per node:
- Hub endpoint IP in:
  - `loki.write` URL
  - `prometheus.remote_write` URL
- Instance label:
  - `loki.process.stage.static_labels.instance`
  - `prometheus.relabel.rule.replacement`
- Environment label in relabel rules if needed
- Timezone in `stage.timestamp.location`
- Laravel log glob path under `local.file_match "laravel_logs"`

### 4.3 Update `docker-compose.yml` bind mount path

Set `LARAVEL_LOG_DIR` to your real host path. Default is Forge-style:
- `/home/forge/app/storage/logs` on host -> `/var/log/laravel` in container

Ensure `config.alloy` keeps the log glob as:
- `__path__ = "/var/log/laravel/*.log"`

Use `.env` for per-node values:

```bash
cd /opt/monitoring/alloy-docker
cp .env.example .env
nano .env
```

Set:
- `HOSTNAME=<INSTANCE_NAME>`
- `LARAVEL_LOG_DIR=/home/forge/<SITE_NAME>/storage/logs`

### 4.4 Start stack

```bash
cd /opt/monitoring/alloy-docker
docker compose up -d
docker compose ps
docker compose logs -f --tail=80
```

Expected containers:
- `alloy`
- `nginx-exporter`
- `phpfpm-exporter`

### 4.5 Verify Alloy UI and exporters

```bash
curl -s http://127.0.0.1:12345/ | head -1
curl -s http://127.0.0.1:9113/metrics | head -5
curl -s http://127.0.0.1:9253/metrics | head -5
```

If `phpfpm-exporter` has no metrics, check its scrape URI format and container logs:

```bash
docker compose logs --tail=80 phpfpm-exporter
```

## 5. Laravel Cleanup

Run inside Laravel app directory (`LARAVEL_APP_DIR`):

```bash
cd /path/to/laravel-app
composer remove keepsuit/laravel-opentelemetry open-telemetry/sdk open-telemetry/exporter-otlp
```

Then remove any leftover package integration from the Laravel app:
- Delete published OpenTelemetry config files if present, typically `config/opentelemetry.php`
- Remove service provider registration if you added one manually
- Remove any custom trace middleware registration from `bootstrap/app.php` or `app/Http/Kernel.php`
- Remove all `OTEL_*` variables from `.env`, deployment secrets, and process manager config
- Clear cached Laravel config after cleanup

```bash
php artisan optimize:clear
```

## 6. Validation Checklist

Per Laravel node:
- `http://127.0.0.1:8080/nginx_status` returns Nginx status
- `http://127.0.0.1:8080/fpm-status` returns PHP-FPM status
- `docker compose ps` shows agent stack running
- `http://127.0.0.1:12345/` responds (Alloy UI)
- `:9113/metrics` and `:9253/metrics` respond
- Alloy logs show successful remote writes to hub

From monitoring hub:
- Loki query shows node logs (`instance="INSTANCE_NAME"`)
- Mimir query shows node metrics (`up`/node or exporter jobs)

## 7. Operations

Common commands:

```bash
cd /opt/monitoring/alloy-docker
docker compose ps
docker compose logs -f alloy
docker compose restart alloy
docker compose down
docker compose up -d
```

Update config:

```bash
nano /opt/monitoring/alloy-docker/config.alloy
docker compose restart alloy
docker compose logs --tail=50 alloy
```

If migrating from systemd Alloy:

```bash
sudo systemctl stop alloy
sudo systemctl disable alloy
sudo systemctl status alloy
```
