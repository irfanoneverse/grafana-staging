# LGTM Observability Rollout Guide

This guide provides a general, end-to-end flow for deploying a centralized
observability stack for application servers.

The setup uses:

- **Grafana** for dashboards, querying, and visualization
- **Loki** for log storage
- **Mimir** for metrics storage
- **Grafana Alloy** as the collection agent
- **S3** for Loki and Mimir object storage

The recommended topology is:

- One **monitoring hub** EC2 instance runs Grafana, Loki, Mimir, and hub
  self-monitoring.
- One or more **application EC2 instances** run the application normally on the
  host.
- Each application instance runs a small Docker-based monitoring agent stack
  that sends logs and metrics to the monitoring hub.

> The application itself does not need to be containerized. Docker is used only
> for the observability components.

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Required Inputs](#2-required-inputs)
3. [AWS Foundation](#3-aws-foundation)
4. [Monitoring Hub Setup](#4-monitoring-hub-setup)
5. [Application Node Setup](#5-application-node-setup)
6. [Validation](#6-validation)
7. [Operations](#7-operations)
8. [Troubleshooting](#8-troubleshooting)
9. [Rollout Checklist](#9-rollout-checklist)

## 1. Architecture Overview

### 1.1 Data Flow

```text
Application logs
Nginx metrics
PHP-FPM metrics
Host metrics
        |
        v
Grafana Alloy on application EC2
        |
        |-- logs ----------> Loki on monitoring hub
        |
        `-- metrics -------> Mimir on monitoring hub

Grafana on monitoring hub reads from Loki and Mimir.
```

### 1.2 Network Model

Application nodes only need outbound access to the monitoring hub.

The monitoring hub should allow inbound:

| Port       | Source                            | Purpose                 |
| ---------- | --------------------------------- | ----------------------- |
| `80/tcp`   | Office, VPN, or trusted CIDR      | Grafana UI              |
| `3100/tcp` | Application server security group | Loki log ingestion      |
| `9009/tcp` | Application server security group | Mimir metrics ingestion |
| `22/tcp`   | Office, VPN, or trusted CIDR      | SSH administration      |

Application nodes do not need extra inbound rules from the monitoring hub.

## 2. Required Inputs

Prepare these values before starting.

### 2.1 AWS Values

| Variable               | Example                      | Notes                            |
| ---------------------- | ---------------------------- | -------------------------------- |
| `AWS_REGION`           | `ap-southeast-1`             | Region for EC2 and S3            |
| `OBSERVABILITY_BUCKET` | `company-observability-prod` | S3 bucket used by Loki and Mimir |
| `AWS_ACCOUNT_ID`       | `123456789012`               | Needed for IAM policy attachment |
| `HUB_INSTANCE_ID`      | `i-xxxxxxxxxxxxxxxxx`        | Monitoring hub EC2 instance ID   |

### 2.2 Monitoring Hub Values

| Variable                 | Example                      | Notes                             |
| ------------------------ | ---------------------------- | --------------------------------- |
| `HUB_PRIVATE_IP`         | `10.0.10.25`                 | Used by app nodes to send data    |
| `GRAFANA_ROOT_URL`       | `http://grafana.example.com` | Public or private URL for Grafana |
| `GRAFANA_ADMIN_PASSWORD` | `use-a-strong-password`      | Change before production use      |

### 2.3 Application Node Values

Define these per application server.

| Variable        | Example                                | Notes                         |
| --------------- | -------------------------------------- | ----------------------------- |
| `INSTANCE_NAME` | `api-prod-01`                          | Unique label shown in Grafana |
| `ENVIRONMENT`   | `production`                           | Environment label             |
| `TZ_NAME`       | `Asia/Kuala_Lumpur`                    | IANA timezone for log parsing |
| `APP_LOG_DIR`   | `/home/forge/example.com/storage/logs` | Host path to application logs |
| `APP_DIR`       | `/home/forge/example.com`              | Application root directory    |

## 3. AWS Foundation

Complete this section once before deploying the monitoring hub.

### 3.1 Create the S3 Bucket

Create one private S3 bucket for Loki and Mimir storage.

```bash
aws s3api create-bucket \
  --bucket OBSERVABILITY_BUCKET \
  --region AWS_REGION \
  --create-bucket-configuration LocationConstraint=AWS_REGION

aws s3api put-bucket-versioning \
  --bucket OBSERVABILITY_BUCKET \
  --versioning-configuration Status=Enabled

aws s3api put-public-access-block \
  --bucket OBSERVABILITY_BUCKET \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws s3api put-bucket-encryption \
  --bucket OBSERVABILITY_BUCKET \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]
  }'
```

Replace `OBSERVABILITY_BUCKET` and `AWS_REGION` with your real values.

### 3.2 Configure S3 Lifecycle Retention

Use a lifecycle policy to control storage cost and retention.

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket OBSERVABILITY_BUCKET \
  --lifecycle-configuration file://aws/s3-lifecycle-policy.json
```

Recommended retention pattern:

| Age           | Storage Action                 |
| ------------- | ------------------------------ |
| `0-30 days`   | Keep in S3 Standard            |
| `30-100 days` | Move to S3 Intelligent-Tiering |
| `100+ days`   | Delete                         |

Adjust this policy for your retention requirements.

### 3.3 Create the Monitoring Hub IAM Role

The monitoring hub should access S3 through an EC2 instance profile. Do not put
static AWS credentials on the server.

Before creating the IAM policy, update `aws/iam-policy-lgtm-s3.json` so the
bucket ARN matches `OBSERVABILITY_BUCKET`.

```bash
aws iam create-policy \
  --policy-name LGTMStackS3Access \
  --policy-document file://aws/iam-policy-lgtm-s3.json
```

Create the EC2 trust policy:

```bash
cat > /tmp/ec2-trust-policy.json << 'EOJSON'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOJSON
```

Create and attach the role:

```bash
aws iam create-role \
  --role-name LGTMServerRole \
  --assume-role-policy-document file:///tmp/ec2-trust-policy.json

aws iam attach-role-policy \
  --role-name LGTMServerRole \
  --policy-arn arn:aws:iam::AWS_ACCOUNT_ID:policy/LGTMStackS3Access

aws iam create-instance-profile \
  --instance-profile-name LGTMServerProfile

aws iam add-role-to-instance-profile \
  --instance-profile-name LGTMServerProfile \
  --role-name LGTMServerRole

aws ec2 associate-iam-instance-profile \
  --instance-id HUB_INSTANCE_ID \
  --iam-instance-profile Name=LGTMServerProfile
```

Application servers do not need S3 access for this design.

### 3.4 Configure Security Groups

Use two security group roles:

- A **monitoring hub security group** attached to the hub EC2.
- An **application security group** attached to application EC2 instances.

Monitoring hub inbound rules:

| Type       | Port   | Source                       |
| ---------- | ------ | ---------------------------- |
| HTTP       | `80`   | Office, VPN, or trusted CIDR |
| Custom TCP | `3100` | Application security group   |
| Custom TCP | `9009` | Application security group   |
| SSH        | `22`   | Office, VPN, or trusted CIDR |

Application server inbound rules:

- Keep the existing application and SSH rules.
- No additional monitoring inbound rule is required.

## 4. Monitoring Hub Setup

Run this section on the EC2 instance that will host Grafana, Loki, and Mimir.

### 4.1 Install Docker

```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
sudo apt install -y docker-compose-plugin
```

Log out and log back in so the Docker group change takes effect.

Verify Docker:

```bash
docker --version
docker compose version
docker run --rm hello-world
```

### 4.2 Copy the Repository

```bash
cd /opt
sudo mkdir -p grafana-main-hub && sudo chown $USER:$USER grafana-main-hub
git clone YOUR_REPOSITORY_URL
cd /opt/grafana-main-hub
```

If Git access is not available from the server, copy the required files by
another deployment method.

Required hub directory:

```text
main-server/
```

### 4.3 Review Hub Configuration

Before starting the stack, confirm these files match your environment:

| File                                                            | What to Check                       |
| --------------------------------------------------------------- | ----------------------------------- |
| `main-server/docker-compose.yml`                                | Grafana admin password and root URL |
| `main-server/loki-config.yaml`                                  | S3 bucket name and region           |
| `main-server/mimir-config.yaml`                                 | S3 bucket name and region           |
| `main-server/grafana/provisioning/datasources/datasources.yaml` | Loki and Mimir datasource URLs      |

In `main-server/docker-compose.yml`, change these values before production use:

```yaml
GF_SECURITY_ADMIN_PASSWORD=changeme
GF_SERVER_ROOT_URL=http://localhost:3000
```

### 4.4 Start the Hub Stack

```bash
cd /opt/lgtm/main-server
docker compose up -d
docker compose ps
docker compose logs -f --tail=80
```

Expected containers:

- `grafana`
- `loki`
- `mimir`
- `alloy-hub`

### 4.5 Verify Hub Services

```bash
curl -s http://127.0.0.1/api/health
curl -s http://127.0.0.1:3100/ready
curl -s http://127.0.0.1:9009/ready
```

Open Grafana in a browser:

```text
http://HUB_PUBLIC_IP_OR_DNS/
```

## 5. Application Node Setup

Repeat this section for every application EC2 instance that should be monitored.

### 5.1 Install Docker

```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
sudo apt install -y docker-compose-plugin
```

Log out and log back in, then verify:

```bash
docker --version
docker compose version
docker run --rm hello-world
```

### 5.2 Enable Nginx and PHP-FPM Status Endpoints

The monitoring agent reads local status endpoints from the application host.
These endpoints should only be available on localhost.

Enable PHP-FPM status:

```bash
sudo sed -i 's#^;*pm.status_path = .*#pm.status_path = /fpm-status#' /etc/php/*/fpm/pool.d/www.conf
sudo systemctl restart php*-fpm
```

Check the active PHP-FPM socket:

```bash
ls /run/php/
```

Create an Nginx localhost status server:

```bash
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
```

If your socket name is different, update this line before reloading Nginx:

```nginx
fastcgi_pass unix:/run/php/php-fpm.sock;
```

Apply the Nginx config:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Verify local endpoints:

```bash
curl -s http://127.0.0.1:8080/nginx_status
curl -s http://127.0.0.1:8080/fpm-status
```

### 5.3 Copy the Agent Files

```bash
cd /opt
sudo mkdir -p monitoring && sudo chown $USER:$USER monitoring
git clone YOUR_REPOSITORY_URL monitoring
cd /opt/monitoring
```

Required agent directory:

```text
alloy-docker/
```

### 5.4 Configure the Agent

Create the node-specific environment file:

```bash
cd /opt/monitoring/alloy-docker
cp .env.example .env
nano .env
```

Set:

```env
HOSTNAME=INSTANCE_NAME
LARAVEL_LOG_DIR=APP_LOG_DIR
```

The variable name `LARAVEL_LOG_DIR` is used by the Docker Compose file. It can
point to any application log directory, not only Laravel.

Edit Alloy configuration:

```bash
nano /opt/monitoring/alloy-docker/config.alloy
```

Update the following values:

| Setting                | Required Value                                |
| ---------------------- | --------------------------------------------- |
| Loki write URL         | `http://HUB_PRIVATE_IP:3100/loki/api/v1/push` |
| Mimir remote write URL | `http://HUB_PRIVATE_IP:9009/api/v1/push`      |
| Instance label         | `INSTANCE_NAME`                               |
| Environment label      | `ENVIRONMENT`                                 |
| Timestamp timezone     | `TZ_NAME`                                     |
| Application log path   | Container path used by Compose                |

The Compose file mounts the host log directory into the container. Keep the
Alloy log glob aligned with the container path:

```text
/host/logs/laravel/*.log
```

### 5.5 Start the Agent Stack

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

### 5.6 Verify Local Agent Services

```bash
curl -s http://127.0.0.1:12345/ | head -1
curl -s http://127.0.0.1:9113/metrics | head -5
curl -s http://127.0.0.1:9253/metrics | head -5
```

If the PHP-FPM exporter does not return metrics, check:

```bash
docker compose logs --tail=80 phpfpm-exporter
curl -s http://127.0.0.1:8080/fpm-status
```

### DONE
