# Re-deploy Agent Stack — Quick Guide

## Variables

Change these per node before running any steps:

```bash
export INSTANCE_NAME="sso-api-staging"
export HUB_IP="172.31.27.45"
export LARAVEL_LOG_DIR="/data/sso-api/storage/logs"
export ENVIRONMENT="staging"
export TZ_NAME="Asia/Kuala_Lumpur"
```

---

## Step 1 — Bring Down Existing Stack

```bash
cd /opt/grafana-nodes/alloy-docker
docker compose down
docker ps
```

---

## Step 2 — Remove Old Folder

```bash
rm -rf /opt/grafana-nodes
ls /opt
```

---

## Step 3 — Pull Fresh from GitHub

```bash
cd /opt
mkdir grafana-nodes
cd grafana-nodes
git init
git remote add origin https://github.com/irfanoneverse/grafana-staging.git
git fetch origin sso-api-staging
git checkout origin/sso-api-staging -- alloy-docker
ls /opt/grafana-nodes/alloy-docker
```

---

## Step 4 — Configure `.env`

```bash
cd /opt/grafana-nodes/alloy-docker
cp .env.example .env
nano .env
```

---

## Step 5 — Start Stack

```bash
docker compose up -d
docker compose ps
docker compose logs --tail=80 alloy
```

---

## Step 6 — AWS Security Group

Add the following inbound rules to the **hub's security group**:

| Port | Protocol | Source                               |
| ---- | -------- | ------------------------------------ |
| 3100 | TCP      | Client private IP `/32` or client SG |
| 9009 | TCP      | Client private IP `/32` or client SG |

Then verify connectivity from the client EC2:

```bash
nc -vz 172.31.27.45 3100
nc -vz 172.31.27.45 9009
```

Both should return `Connection succeeded`.
