# Ubuntu EC2 - Remove OpenTelemetry (Keep Grafana Alloy)

This is a simplified cleanup guide for Ubuntu EC2 nodes where OpenTelemetry was used before, and now only Grafana Alloy is used.

## 1) Check what OpenTelemetry footprint still exists

```bash
sudo systemctl list-unit-files | grep -Ei 'otel|opentelemetry|collector'
docker ps --format '{{.Names}} {{.Image}}' | grep -Ei 'otel|opentelemetry|collector'
ps aux | grep -Ei 'otel|opentelemetry|collector' | grep -v grep
```

## 2) Remove OpenTelemetry from Laravel app

Run inside your Laravel app directory:

```bash
cd /path/to/laravel-app
composer remove keepsuit/laravel-opentelemetry open-telemetry/sdk open-telemetry/exporter-otlp
rm -f config/opentelemetry.php
```

### 2a) Check for remaining OTel references in app code

Even after removing packages, middleware or other files may still import OTel classes and cause a 500 on every request. Scan for them:

```bash
grep -r "opentelemetry\|OpenTelemetry\|keepsuit" config/ bootstrap/ app/ 2>/dev/null
```

If any files appear (e.g. `app/Http/Middleware/TraceIdMiddleware.php`), remove the OTel dependency from them. For a middleware that only injected a trace ID into logs, replace the entire file with a safe no-op:

```bash
cat > app/Http/Middleware/TraceIdMiddleware.php << 'EOF'
<?php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class TraceIdMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        return $next($request);
    }
}
EOF
```

### 2b) Remove OTEL\_\* variables from .env

```bash
sed -i '/^OTEL_/d' /path/to/laravel-app/.env
```

Also remove from:

- deployment secrets / CI variables
- process manager configs (if any)

### 2c) Fix storage permissions (if composer was run as root)

Running `composer` as root changes ownership of storage files, causing PHP-FPM to fail writing logs. Fix with:

```bash
chown -R www-data:www-data /path/to/laravel-app/storage
chown -R www-data:www-data /path/to/laravel-app/bootstrap/cache
```

### 2d) Clear Laravel caches

```bash
php artisan optimize:clear
```

> **Note:** If `artisan` itself throws an OTel class error, it means the config cache still references the deleted config. Manually delete the cache files first:
>
> ```bash
> rm -f bootstrap/cache/config.php
> rm -f bootstrap/cache/packages.php
> rm -f bootstrap/cache/services.php
> ```
>
> Then re-run `php artisan optimize:clear`.

## 3) Stop and disable OpenTelemetry services

```bash
sudo systemctl stop otelcol otel-collector aws-otel-collector 2>/dev/null || true
sudo systemctl disable otelcol otel-collector aws-otel-collector 2>/dev/null || true
sudo systemctl daemon-reload
```

## 4) Uninstall OpenTelemetry packages (Ubuntu)

```bash
sudo apt update
sudo apt remove -y otelcol otelcol-contrib aws-otel-collector || true
sudo apt autoremove -y
```

## 5) Delete leftover OpenTelemetry files (if present)

```bash
sudo rm -rf /etc/otelcol* /etc/otel-collector* /opt/aws/aws-otel-collector /var/lib/otelcol* /var/log/otel* 2>/dev/null || true
```

## 6) Remove old OpenTelemetry Docker containers/images (if used before)

```bash
docker ps -a --format '{{.ID}} {{.Names}} {{.Image}}' | grep -Ei 'otel|opentelemetry|collector'
```

If any appear, remove them explicitly.

## 7) Verify only Grafana Alloy path is active

```bash
cd /opt/monitoring/alloy-docker
docker compose ps
docker compose logs --tail=100 alloy
curl -s http://127.0.0.1:12345/ | head -1
```

If Alloy is healthy and no OpenTelemetry service/process/container remains, cleanup is complete.
