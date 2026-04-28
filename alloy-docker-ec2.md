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
- `CLIENT_PRIVATE_IP` = Laravel client EC2 private IP
- `INSTANCE_NAME` = unique node label (example: `api-prod-01`)
- `ENVIRONMENT` = `production`/`staging`
- `TZ_NAME` = IANA timezone (example: `Asia/Kuala_Lumpur`)
- `LARAVEL_LOG_DIR` = Laravel logs directory on that node
- `LARAVEL_APP_DIR` = Laravel project root on that node

For the `sso-api-staging` EC2, use:

```bash
export HUB_IP="172.31.27.45"
export CLIENT_PRIVATE_IP="172.31.2.217"
export INSTANCE_NAME="sso-api-staging"
export ENVIRONMENT="staging"
export TZ_NAME="Asia/Kuala_Lumpur"
export LARAVEL_LOG_DIR="/data/sso-api/storage/logs"
export LARAVEL_APP_DIR="/data/sso-api"
```

Notes:
- `18.136.17.219` is the public IP for SSH/admin access to `sso-api-staging`.
- `172.31.2.217` is the private IP of the `sso-api-staging` client EC2.
- Use the monitoring hub private IP for `HUB_IP` when both EC2 instances are in the same VPC.
- The repository default currently keeps `HUB_IP=172.31.27.45`; change it if your hub private IP is different.
- Do not set `HUB_IP` to `172.31.2.217` unless the monitoring hub also runs on this same EC2.

Confirm the real paths:

```bash
ls -lah "$LARAVEL_APP_DIR"
ls -lah "$LARAVEL_LOG_DIR"
ls -lah /var/log/nginx
ls -lah /var/log | grep fpm
```

## 2. Prerequisites

From [aws-setup-docker-monitoringhub.md](aws-setup-docker-monitoringhub.md), ensure:
- Hub is healthy (`docker compose up -d` completed on hub)
- Hub SG allows inbound from app SG on `3100`, `9009`
- App node has outbound access to hub
- You know the hub private IP or private DNS name

Check network access from the Laravel EC2:

```bash
nc -vz "$HUB_IP" 3100
nc -vz "$HUB_IP" 9009
```

If `nc` is not installed:

```bash
sudo apt update
sudo apt install -y netcat-openbsd
```

Install Docker on each Laravel EC2:

```bash
sudo apt update
sudo apt upgrade -y
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
sudo apt install -y docker-compose-plugin git
```

Log out and log back in so the `docker` group applies:

```bash
exit
```

Verify after login:

```bash
docker --version
docker compose version
docker run --rm hello-world
```

## 3. Enable Host Status Endpoints

Alloy scrapes metrics from local exporters. The exporters need local Nginx and PHP-FPM status endpoints.

### 3.1 Enable PHP-FPM status path

Find the active PHP-FPM pool:

```bash
ls -lah /etc/php/*/fpm/pool.d/www.conf
grep -R "listen =" /etc/php/*/fpm/pool.d/www.conf
```

Enable `pm.status_path`:

```bash
sudo sed -i 's#^;*pm.status_path = .*#pm.status_path = /fpm-status#' /etc/php/*/fpm/pool.d/www.conf
sudo systemctl restart php*-fpm
```

Confirm PHP-FPM is running:

```bash
systemctl --no-pager --type=service | grep php | grep fpm
ls -lah /run/php/
```

### 3.2 Add Nginx status server (`:8080`)

Check the active PHP-FPM socket:

```bash
ls /run/php/
```

Common examples:
- `/run/php/php8.2-fpm.sock`
- `/run/php/php8.3-fpm.sock`
- `/run/php/php-fpm.sock`

Set the socket path:

```bash
export PHP_FPM_SOCK="/run/php/php8.2-fpm.sock"
```

Create the local-only Nginx status server:

```bash
sudo tee /etc/nginx/conf.d/stub_status.conf > /dev/null << EONGINX
server {
    listen 8080;
    server_name localhost;

    location = /nginx_status {
        stub_status on;
        allow 127.0.0.1;
        deny all;
    }

    location = /fpm-status {
        fastcgi_pass unix:$PHP_FPM_SOCK;
        fastcgi_param SCRIPT_NAME     /fpm-status;
        fastcgi_param SCRIPT_FILENAME /fpm-status.php;
        fastcgi_param REQUEST_URI     /fpm-status;
        fastcgi_param QUERY_STRING    \$query_string;
        fastcgi_param REQUEST_METHOD  \$request_method;
        allow 127.0.0.1;
        deny all;
    }
}
EONGINX

sudo nginx -t
sudo systemctl reload nginx
```

This server listens only on port `8080` and only allows `127.0.0.1`. It is for local metrics scraping, not public access.

### 3.3 Verify endpoints

```bash
curl -s http://127.0.0.1:8080/nginx_status
curl -s http://127.0.0.1:8080/fpm-status
```

Expected:
- `nginx_status` returns active connections and request counters.
- `fpm-status` returns PHP-FPM pool/process status.

If `fpm-status` returns `502`, the socket path in `/etc/nginx/conf.d/stub_status.conf` is wrong. Update `fastcgi_pass`, then run:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## 4. Deploy Agent Stack

### 4.1 Copy files to node

Clone the repository:

```bash
cd /opt
sudo mkdir -p monitoring
sudo chown "$USER:$USER" monitoring
git clone https://github.com/irfanoneverse/grafana.git monitoring
cd /opt/monitoring
git checkout sso-api-staging
cd /opt/monitoring/alloy-docker
```

Alternative if you only want to copy agent files:

```bash
scp -r alloy-docker/ ubuntu@LARAVEL_EC2_PRIVATE_IP:/opt/monitoring/
```

### 4.2 Create `.env`

The checked-in defaults are already for `sso-api-staging`. Create the runtime `.env` file:

```bash
cd /opt/monitoring/alloy-docker
cp .env.example .env
nano .env
```

Confirm these values:

```env
HOSTNAME=sso-api-staging
INSTANCE_NAME=sso-api-staging
ENVIRONMENT=staging
TZ_NAME=Asia/Kuala_Lumpur
LARAVEL_LOG_DIR=/data/sso-api/storage/logs
HUB_IP=172.31.27.45
```

Change only `HUB_IP` if your monitoring hub private IP is not `172.31.27.45`.

The current `config.alloy` reads Laravel logs from the container path:

```text
/host/logs/laravel/*.log
```

`docker-compose.yml` maps your host path into that container path:

```text
/data/sso-api/storage/logs -> /host/logs/laravel
```

Validate the mount source exists:

```bash
test -d "$LARAVEL_LOG_DIR" && ls -lah "$LARAVEL_LOG_DIR" | head
```

### 4.3 Start stack

```bash
cd /opt/monitoring/alloy-docker
docker compose up -d
docker compose ps
docker compose logs --tail=80 alloy
```

Expected containers:
- `alloy`
- `nginx-exporter`
- `phpfpm-exporter`

### 4.4 Verify Alloy UI and exporters

```bash
curl -s http://127.0.0.1:12345/ | head -1
curl -s http://127.0.0.1:9113/metrics | head -5
curl -s http://127.0.0.1:9253/metrics | head -5
```

If `phpfpm-exporter` has no metrics, check its scrape URI format and container logs:

```bash
docker compose logs --tail=80 phpfpm-exporter
curl -s http://127.0.0.1:8080/fpm-status
```

If `nginx-exporter` has no metrics:

```bash
docker compose logs --tail=80 nginx-exporter
curl -s http://127.0.0.1:8080/nginx_status
```

### 4.5 Verify Data on Monitoring Hub

From the Laravel node, confirm the hub ports are reachable:

```bash
nc -vz "$HUB_IP" 3100
nc -vz "$HUB_IP" 9009
```

From Grafana Explore on the hub:

Loki queries:

```logql
{instance="sso-api-staging"}
{job="laravel", environment="staging"}
{job="nginx"}
```

Mimir/Prometheus queries:

```promql
up
up{instance="sso-api-staging"}
node_uname_info{instance="sso-api-staging"}
nginx_connections_active{instance="sso-api-staging"}
phpfpm_up{instance="sso-api-staging"}
```

Replace `sso-api-staging` with your `INSTANCE_NAME` for other nodes.

## 5. Laravel Cleanup

Run this only if the Laravel app previously used OpenTelemetry packages and you are replacing that setup with Alloy-based logs/metrics collection.

Run inside Laravel app directory (`LARAVEL_APP_DIR`):

```bash
cd "$LARAVEL_APP_DIR"
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

If `composer remove` says a package is not installed, continue with the manual cleanup checks.

## 6. Validation Checklist

Per Laravel node:
- `http://127.0.0.1:8080/nginx_status` returns Nginx status
- `http://127.0.0.1:8080/fpm-status` returns PHP-FPM status
- `docker compose ps` shows agent stack running
- `http://127.0.0.1:12345/` responds (Alloy UI)
- `:9113/metrics` and `:9253/metrics` respond
- Alloy logs show no repeated remote write errors
- Laravel log directory is mounted read-only into the Alloy container

From monitoring hub:
- Loki query shows node logs (`instance="INSTANCE_NAME"`)
- Mimir query shows node metrics (`up`, `node_uname_info`, `nginx_*`, `phpfpm_*`)
- Grafana Explore can query both Loki and Mimir datasources

Fast local validation command:

```bash
cd /opt/monitoring/alloy-docker
docker compose ps
curl -s http://127.0.0.1:8080/nginx_status | head
curl -s http://127.0.0.1:8080/fpm-status | head
curl -s http://127.0.0.1:9113/metrics | head
curl -s http://127.0.0.1:9253/metrics | head
docker compose logs --tail=80 alloy
```

## 7. Operations

Common commands:

```bash
cd /opt/monitoring/alloy-docker
docker compose ps
docker compose logs -f alloy
docker compose logs -f nginx-exporter
docker compose logs -f phpfpm-exporter
docker compose restart alloy
docker compose down
docker compose up -d
```

Update config:

```bash
cd /opt/monitoring/alloy-docker
nano config.alloy
docker compose restart alloy
docker compose logs --tail=50 alloy
```

Update Docker images:

```bash
cd /opt/monitoring/alloy-docker
docker compose pull
docker compose up -d
docker image prune -f
```

If migrating from systemd Alloy:

```bash
sudo systemctl stop alloy
sudo systemctl disable alloy
sudo systemctl status alloy
```

Rollback the Docker agent only:

```bash
cd /opt/monitoring/alloy-docker
docker compose down
```

This stops the monitoring agent containers only. It does not stop Nginx, PHP-FPM, Laravel, or the application database.
