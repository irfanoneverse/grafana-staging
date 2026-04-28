# LGTM Observability Stack - Hub + Multi-EC2 Template

This repository is for a centralized observability setup:
- One EC2 **monitoring hub** runs Grafana + Loki + Mimir.
- Multiple Laravel EC2 nodes run a Dockerized Alloy agent.
- Each Laravel node pushes logs and metrics to the hub.

Use this file for hub and AWS foundation.
Use [alloy-docker-ec2.md](alloy-docker-ec2.md) for per-Laravel-node agent deployment.

## Table of Contents

1. [Architecture](#1-architecture)
2. [AWS Prerequisites](#2-aws-prerequisites)
3. [S3 Lifecycle Policy](#3-s3-lifecycle-policy)
4. [Monitoring Hub EC2 Setup](#4-monitoring-hub-ec2-setup)
5. [Laravel Node Template Handoff](#5-laravel-node-template-handoff)

## 1. Architecture

Data flow:
- Laravel app logs + Nginx logs + PHP-FPM logs -> Alloy -> Loki
- Node/exporter metrics -> Alloy -> Mimir
- Grafana on hub reads Loki/Mimir for unified view

Network model:
- Laravel nodes only need outbound access to hub ports: `3100`, `9009`.
- Hub exposes Grafana UI on host port `80` (current compose mapping `80:3000`).
- Loki listens on `3100`; Mimir listens on `9009`; Grafana listens on `80`.
- Use private IPs/security groups for agent-to-hub traffic whenever the nodes are in the same VPC.

Recommended EC2 layout:
- Monitoring hub EC2: Docker only, runs `main-server/docker-compose.yml`.
- Laravel app EC2 nodes: existing Nginx/PHP-FPM/Laravel stays on host; Docker runs only Alloy/exporters.
- One private S3 bucket stores Loki and Mimir data.

## 2. AWS Prerequisites

Before running commands, set these values in your shell:

```bash
export AWS_REGION="ap-southeast-1"
export OBS_BUCKET="YOUR_COMPANY-observability-data"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export HUB_INSTANCE_ID="i-xxxxxxxxxxxxxxxxx"
export GRAFANA_ROOT_URL="http://MONITORING_HUB_PUBLIC_IP/"
export GRAFANA_ADMIN_PASSWORD="CHANGE_THIS_PASSWORD"
```

Replace:
- `AWS_REGION` with the region where the hub EC2 and bucket should run.
- `OBS_BUCKET` with a globally unique S3 bucket name.
- `HUB_INSTANCE_ID` with the monitoring hub EC2 instance ID.
- `GRAFANA_ROOT_URL` with the hub public URL, internal URL, or DNS name.
- `GRAFANA_ADMIN_PASSWORD` with a strong admin password.

### 2.1 S3 Bucket

Create one private S3 bucket for Loki and Mimir object storage.

For most regions:

```bash
aws s3api create-bucket \
  --bucket "$OBS_BUCKET" \
  --region "$AWS_REGION" \
  --create-bucket-configuration LocationConstraint="$AWS_REGION"
```

For `us-east-1` only, omit `LocationConstraint`:

```bash
aws s3api create-bucket \
  --bucket "$OBS_BUCKET" \
  --region us-east-1
```

Enable versioning, block all public access, and enable default encryption:

```bash
aws s3api put-bucket-versioning \
  --bucket "$OBS_BUCKET" \
  --versioning-configuration Status=Enabled

aws s3api put-public-access-block \
  --bucket "$OBS_BUCKET" \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws s3api put-bucket-encryption \
  --bucket "$OBS_BUCKET" \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]
  }'
```

What these commands do:
- `create-bucket`: creates the S3 bucket used by Loki and Mimir.
- `put-bucket-versioning`: keeps object versions to help recover from accidental overwrite/delete.
- `put-public-access-block`: prevents logs and metrics from accidentally becoming public.
- `put-bucket-encryption`: encrypts every object written to the bucket with S3-managed AES-256.

Verify:

```bash
aws s3api get-bucket-location --bucket "$OBS_BUCKET"
aws s3api get-public-access-block --bucket "$OBS_BUCKET"
aws s3api get-bucket-encryption --bucket "$OBS_BUCKET"
```

### 2.2 IAM Role for Monitoring Hub EC2

Attach S3 access via EC2 instance profile. Do not store static AWS credentials on disk.

Use one policy file only and keep it aligned with the bucket used in:
- `main-server/loki-config.yaml`
- `main-server/mimir-config.yaml`

Create the policy file:

```bash
mkdir -p /tmp/grafana-aws

cat > /tmp/grafana-aws/iam-policy-lgtm-s3.json << EOJSON
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "GrafanaS3Access",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::$OBS_BUCKET",
        "arn:aws:s3:::$OBS_BUCKET/*"
      ]
    }
  ]
}
EOJSON
```

Create the IAM policy:

```bash
aws iam create-policy \
  --policy-name GrafanaS3Access \
  --policy-document file:///tmp/grafana-aws/iam-policy-lgtm-s3.json
```

Create the EC2 trust policy:

```bash
cat > /tmp/grafana-aws/ec2-trust-policy.json << 'EOJSON'
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

Create the role, attach the policy, create the instance profile, and attach it to the hub EC2:

```bash
aws iam create-role \
  --role-name GrafanaServerRole \
  --assume-role-policy-document file:///tmp/grafana-aws/ec2-trust-policy.json

aws iam attach-role-policy \
  --role-name GrafanaServerRole \
  --policy-arn "arn:aws:iam::$AWS_ACCOUNT_ID:policy/GrafanaS3Access"

aws iam create-instance-profile \
  --instance-profile-name LGTMServerProfile

aws iam add-role-to-instance-profile \
  --instance-profile-name LGTMServerProfile \
  --role-name GrafanaServerRole

aws ec2 associate-iam-instance-profile \
  --instance-id "$HUB_INSTANCE_ID" \
  --iam-instance-profile Name=LGTMServerProfile
```

If the EC2 already has an instance profile, use the AWS console or `replace-iam-instance-profile-association` instead of `associate-iam-instance-profile`.

Verify from the monitoring hub EC2:

```bash
aws sts get-caller-identity
aws s3 ls "s3://$OBS_BUCKET"
```

Laravel EC2 app nodes do not need S3 IAM access for this design.

### 2.3 Security Groups

Recommended security groups:
- `grafana-main-hub`: attach to monitoring hub EC2.
- `grafana-client`: attach to Laravel EC2 app nodes.

`grafana-main-hub` inbound:

| Type | Port | Source | Purpose |
| --- | ---: | --- | --- |
| HTTP | `80` | Office/VPN CIDR | Grafana UI |
| SSH | `22` | Office/VPN CIDR | Admin SSH |
| Custom TCP | `3100` | `grafana-client` SG | Loki ingest from Alloy |
| Custom TCP | `9009` | `grafana-client` SG | Mimir remote_write from Alloy |

`grafana-client` inbound:

| Type | Port | Source | Purpose |
| --- | ---: | --- | --- |
| SSH | `22` | Office/VPN CIDR | Admin SSH |
| HTTP/HTTPS/app ports | app-specific | Load balancer/allowed CIDR | Existing application traffic |

`grafana-client` outbound:
- Allow outbound to the hub private IP on `3100/tcp` and `9009/tcp`.
- Default AWS outbound `0.0.0.0/0` also works, but a narrower rule is better.

If using AWS CLI, find the SG IDs first:

```bash
aws ec2 describe-security-groups \
  --filters Name=group-name,Values=grafana-main-hub,grafana-client \
  --query 'SecurityGroups[*].[GroupName,GroupId]' \
  --output table
```

Then add inbound rules to the hub SG:

```bash
export HUB_SG_ID="sg-hubxxxxxxxx"
export APP_SG_ID="sg-appxxxxxxxx"
export OFFICE_CIDR="x.x.x.x/32"

aws ec2 authorize-security-group-ingress \
  --group-id "$HUB_SG_ID" \
  --protocol tcp \
  --port 3100 \
  --source-group "$APP_SG_ID"

aws ec2 authorize-security-group-ingress \
  --group-id "$HUB_SG_ID" \
  --protocol tcp \
  --port 9009 \
  --source-group "$APP_SG_ID"

aws ec2 authorize-security-group-ingress \
  --group-id "$HUB_SG_ID" \
  --protocol tcp \
  --port 80 \
  --cidr "$OFFICE_CIDR"

aws ec2 authorize-security-group-ingress \
  --group-id "$HUB_SG_ID" \
  --protocol tcp \
  --port 22 \
  --cidr "$OFFICE_CIDR"
```

## 3. S3 Lifecycle Policy

Apply lifecycle policy to control cost and retention.

This repository includes:
- `aws/s3-lifecycle-policy.json`
- `main-server/aws/s3-lifecycle-policy.json`

Current recommended retention:
- `0-30 days`: S3 Standard
- `30-100 days`: S3 Intelligent-Tiering
- `100+ days`: Delete

Apply the repository policy:

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket "$OBS_BUCKET" \
  --lifecycle-configuration file://aws/s3-lifecycle-policy.json
```

Or create a 100-day policy directly:

```bash
cat > /tmp/grafana-aws/s3-lifecycle-policy.json << 'EOJSON'
{
  "Rules": [
    {
      "ID": "ObservabilityDataLifecycle",
      "Status": "Enabled",
      "Filter": { "Prefix": "" },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "INTELLIGENT_TIERING"
        }
      ],
      "Expiration": {
        "Days": 100
      }
    }
  ]
}
EOJSON

aws s3api put-bucket-lifecycle-configuration \
  --bucket "$OBS_BUCKET" \
  --lifecycle-configuration file:///tmp/grafana-aws/s3-lifecycle-policy.json
```

Verify:

```bash
aws s3api get-bucket-lifecycle-configuration --bucket "$OBS_BUCKET"
```

## 4. Monitoring Hub EC2 Setup

### 4.1 Host Bootstrap

Run on the monitoring hub EC2:

```bash
sudo apt update
sudo apt upgrade -y
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
sudo apt install -y docker-compose-plugin git awscli
```

Log out and log back in so the `docker` group applies:

```bash
exit
```

After logging in again:

```bash
docker --version
docker compose version
aws sts get-caller-identity
```

### 4.2 Deploy LGTM Stack

Clone the repository:

```bash
cd /opt
sudo mkdir -p grafana-main-hub
sudo chown "$USER:$USER" grafana-main-hub
git clone https://github.com/irfanoneverse/grafana.git grafana-main-hub
cd /opt/grafana-main-hub/main-server
```

Before first start, update these files:
- `main-server/docker-compose.yml`
- `main-server/loki-config.yaml`
- `main-server/mimir-config.yaml`
- `main-server/grafana/provisioning/datasources/datasources.yaml`

Minimum required changes:
- In `docker-compose.yml`, change `GF_SECURITY_ADMIN_PASSWORD=changeme`.
- In `docker-compose.yml`, change `GF_SERVER_ROOT_URL` to the real URL, for example `http://MONITORING_HUB_PUBLIC_IP/`.
- In `loki-config.yaml`, set the S3 `bucketnames`, `region`, and `endpoint`.
- In `mimir-config.yaml`, set the S3 `bucket_name`, `region`, and `endpoint`.

Quick edit commands:

```bash
nano docker-compose.yml
nano loki-config.yaml
nano mimir-config.yaml
```

Or apply the common replacements with commands:

```bash
cp docker-compose.yml docker-compose.yml.bak
cp loki-config.yaml loki-config.yaml.bak
cp mimir-config.yaml mimir-config.yaml.bak

sed -i "s#GF_SECURITY_ADMIN_PASSWORD=.*#GF_SECURITY_ADMIN_PASSWORD=$GRAFANA_ADMIN_PASSWORD # CHANGE THIS IN PRODUCTION#g" docker-compose.yml
sed -i "s#GF_SERVER_ROOT_URL=.*#GF_SERVER_ROOT_URL=$GRAFANA_ROOT_URL # Update to your domain#g" docker-compose.yml

sed -i "s#bucketnames: .*#bucketnames: $OBS_BUCKET#g" loki-config.yaml
sed -i "s#region: .*#region: $AWS_REGION # Change to your region#g" loki-config.yaml
sed -i "s#endpoint: s3\\..*\\.amazonaws\\.com.*#endpoint: s3.$AWS_REGION.amazonaws.com # Change to your region#g" loki-config.yaml

sed -i "s#bucket_name: .*#bucket_name: $OBS_BUCKET#g" mimir-config.yaml
sed -i "s#region: .*#region: $AWS_REGION # Change to your region#g" mimir-config.yaml
sed -i "s#endpoint: s3\\..*\\.amazonaws\\.com.*#endpoint: s3.$AWS_REGION.amazonaws.com # Change to your region#g" mimir-config.yaml
```

Review the final values:

```bash
grep -nE "GF_SECURITY_ADMIN_PASSWORD|GF_SERVER_ROOT_URL" docker-compose.yml
grep -nE "bucketnames|bucket_name|region|endpoint" loki-config.yaml mimir-config.yaml
```

Start the stack:

```bash
docker compose up -d
docker compose ps
docker compose logs --tail=80 loki
docker compose logs --tail=80 mimir
docker compose logs --tail=80 grafana
```

Expected containers:
- `grafana`
- `loki`
- `mimir`
- `alloy-hub`

### 4.3 Core Hub Files

- `main-server/docker-compose.yml`
- `main-server/loki-config.yaml`
- `main-server/mimir-config.yaml`
- `main-server/alertmanager-fallback-config.yaml`
- `main-server/grafana/provisioning/datasources/datasources.yaml`

### 4.4 Hub Validation

Run on the monitoring hub EC2:

```bash
curl -s http://127.0.0.1:80/api/health
curl -s http://127.0.0.1:3100/ready
curl -s http://127.0.0.1:9009/ready
```

Expected:
- Grafana health returns JSON with database status.
- Loki `/ready` returns ready.
- Mimir `/ready` returns ready or a successful readiness response after startup.

Check S3 writes after the stack has been running:

```bash
aws s3 ls "s3://$OBS_BUCKET" --recursive --summarize --human-readable | tail
```

Open Grafana:

```text
http://MONITORING_HUB_PUBLIC_IP/
```

Default login:
- Username: `admin`
- Password: the value you set in `GF_SECURITY_ADMIN_PASSWORD`

In Grafana Explore:
- Select Loki and query `{job="laravel"}` after app nodes are connected.
- Select Mimir/Prometheus and query `up` after app nodes are connected.

### 4.5 Common Hub Operations

```bash
cd /opt/grafana-main-hub/main-server
docker compose ps
docker compose logs -f grafana
docker compose logs -f loki
docker compose logs -f mimir
docker compose restart grafana
docker compose pull
docker compose up -d
```

If Loki or Mimir cannot access S3:

```bash
aws sts get-caller-identity
aws s3 ls "s3://$OBS_BUCKET"
docker compose logs --tail=120 loki
docker compose logs --tail=120 mimir
```

Check these first:
- EC2 instance profile is attached to the hub EC2.
- IAM policy bucket ARN matches the actual bucket.
- `loki-config.yaml` and `mimir-config.yaml` region/bucket match the actual bucket.
- The hub EC2 has network access to the regional S3 endpoint.

## 5. Laravel Node Template Handoff

Use [alloy-docker-ec2.md](alloy-docker-ec2.md) as the deployment template for each Laravel EC2.

Per-node values that must be unique:
- Laravel log path (Forge path differs per site/server)
- Instance label/name
- Environment label
- Timezone
- Hub private IP/FQDN

Expected result after rollout:
- All nodes visible in Grafana Explore (Loki/Mimir)
- Loki queries return logs by `instance`, `environment`, `job`, and `channel`
- Mimir queries return metrics by `instance`, `environment`, and exporter job

Useful handoff values:

```bash
export HUB_PRIVATE_IP="10.x.x.x"
export LOKI_PUSH_URL="http://$HUB_PRIVATE_IP:3100/loki/api/v1/push"
export MIMIR_REMOTE_WRITE_URL="http://$HUB_PRIVATE_IP:9009/api/v1/push"
```
