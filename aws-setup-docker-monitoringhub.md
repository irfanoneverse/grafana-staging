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

## 2. AWS Prerequisites

### 2.1 S3 Bucket

Create one S3 bucket for Loki and Mimir object storage.

```bash
aws s3api create-bucket \
  --bucket YOUR_OBSERVABILITY_BUCKET \
  --region ap-southeast-1 \
  --create-bucket-configuration LocationConstraint=ap-southeast-1

aws s3api put-bucket-versioning \
  --bucket YOUR_OBSERVABILITY_BUCKET \
  --versioning-configuration Status=Enabled

aws s3api put-public-access-block \
  --bucket YOUR_OBSERVABILITY_BUCKET \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws s3api put-bucket-encryption \
  --bucket YOUR_OBSERVABILITY_BUCKET \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]
  }'
```

### 2.2 IAM Role for Monitoring Hub EC2

Attach S3 access via EC2 instance profile (no static credentials on disk).

Use one policy file only and keep it aligned with the bucket used in:
- `main-server/loki-config.yaml`
- `main-server/mimir-config.yaml`

Policy template file:
- `aws/iam-policy-lgtm-s3.json`

```bash
aws iam create-policy \
  --policy-name LGTMStackS3Access \
  --policy-document file://aws/iam-policy-lgtm-s3.json

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

aws iam create-role \
  --role-name LGTMServerRole \
  --assume-role-policy-document file:///tmp/ec2-trust-policy.json

aws iam attach-role-policy \
  --role-name LGTMServerRole \
  --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/LGTMStackS3Access

aws iam create-instance-profile --instance-profile-name LGTMServerProfile
aws iam add-role-to-instance-profile \
  --instance-profile-name LGTMServerProfile \
  --role-name LGTMServerRole

aws ec2 associate-iam-instance-profile \
  --instance-id i-0xxxxxxxxxxLGTM \
  --iam-instance-profile Name=LGTMServerProfile
```

Laravel EC2 app nodes do not need S3 IAM access for this design.

### 2.3 Security Groups

`ov-aws-grafana` (attach to monitoring hub EC2) inbound:
- `3100/tcp` from Laravel SG (`ov-aws-app`) for Loki
- `9009/tcp` from Laravel SG (`ov-aws-app`) for Mimir
- `80/tcp` from office/VPN CIDR for Grafana UI (matches current compose)
- `22/tcp` from office/VPN CIDR for SSH

`ov-aws-app` (attach to Laravel EC2 app nodes) inbound:
- app traffic and SSH only; no extra inbound from hub required

## 3. S3 Lifecycle Policy

Apply lifecycle policy to control cost and retention:

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket YOUR_OBSERVABILITY_BUCKET \
  --lifecycle-configuration file://aws/s3-lifecycle-policy.json
```

Recommended:
- `0-30 days`: S3 Standard
- `30-100 days`: S3 Intelligent-Tiering
- `100+ days`: Delete

## 4. Monitoring Hub EC2 Setup

### 4.1 Host Bootstrap

```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
sudo apt install -y docker-compose-plugin

docker --version
docker compose version
exit
```

### 4.2 Deploy LGTM Stack

```bash
cd /opt
sudo mkdir -p lgtm && sudo chown $USER:$USER lgtm
git clone YOUR_REPOSITORY_URL lgtm
# example: git clone git@github.com:your-org/your-repo.git lgtm
# or: scp -r main-server/ user@monitoring-hub:/opt/lgtm/

cd /opt/lgtm/main-server
docker compose up -d
docker compose logs -f --tail=80
```

Before first start, confirm these values are correct in config files:
- S3 bucket name and region
- Grafana admin password (`GF_SECURITY_ADMIN_PASSWORD`)
- Grafana root URL (`GF_SERVER_ROOT_URL`) for your real URL/IP

### 4.3 Core Hub Files

- `main-server/docker-compose.yml`
- `main-server/loki-config.yaml`
- `main-server/mimir-config.yaml`
- `main-server/alertmanager-fallback-config.yaml`
- `main-server/grafana/provisioning/datasources/datasources.yaml`

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
