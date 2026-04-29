Re-deploy Agent Stack — Quick Guide
Variables (change per node)
bashexport INSTANCE_NAME="sso-api-staging"
export HUB_IP="172.31.27.45"
export LARAVEL_LOG_DIR="/data/sso-api/storage/logs"
export ENVIRONMENT="staging"
export TZ_NAME="Asia/Kuala_Lumpur"

Step 1 — Bring Down Existing Stack
bashcd /opt/grafana-nodes/alloy-docker
docker compose down
docker ps

Step 2 — Remove Old Folder
bashrm -rf /opt/grafana-nodes
ls /opt

Step 3 — Pull Fresh from GitHub
bashcd /opt
mkdir grafana-nodes
cd grafana-nodes
git init
git remote add origin https://github.com/irfanoneverse/grafana-staging.git
git fetch origin sso-api-staging
git checkout origin/sso-api-staging -- alloy-docker
ls /opt/grafana-nodes/alloy-docker

Step 4 — Configure .env
bashcd /opt/grafana-nodes/alloy-docker
cp .env.example .env
nano .env

Step 5 — Start Stack
bashdocker compose up -d
docker compose ps
docker compose logs --tail=80 alloy

Step 6 — AWS Security Group
On the hub's security group, add inbound rules:
PortProtocolSource3100TCPclient private IP /32 or client SG9009TCPclient private IP /32 or client SG
Verify connection from client EC2:
bashnc -vz 172.31.27.45 3100
nc -vz 172.31.27.45 9009
Both should return Connection succeeded.
