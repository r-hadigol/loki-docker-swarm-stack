Loki Logging Stack on Docker Swarm
Executive Summary
This project implements a logging platform based on Grafana Loki on Docker Swarm. The stack is designed as a multi-component, production-oriented lab architecture for collecting, storing, querying, and visualizing Docker container logs using Loki, Grafana, Alloy, MinIO, Consul, and Nginx.
Loki is deployed in split mode. Core Loki components such as Distributor, Ingester, Querier, Query Frontend, Query Scheduler, Index Gateway, and Compactor run as separate Docker Swarm services. Consul is used as the KV store for Loki rings and internal coordination. MinIO is used as an S3-compatible object store.
In the current version, Alloy runs only on the main host and collects Docker container logs from that host. Logs are pushed through the Nginx Gateway to Loki. Grafana also uses the same Gateway to query Loki.

Project Goals
The main goals of this project are:
·	Deploy Loki on Docker Swarm using split architecture.
·	Use Docker Stack instead of multiple standalone docker-compose files.
·	Manage the full logging stack from a central stack file.
·	Use Docker node labels for controlled service placement.
·	Configure Ingester replication with dedicated WAL volumes.
·	Use Consul for Loki ring and KV storage.
·	Use MinIO as S3-compatible object storage.
·	Add an Nginx Gateway to separate write and read traffic.
·	Collect real Docker logs with Grafana Alloy.
·	Clean up Loki labels to make Grafana Explore easier to use.
·	Prepare the stack for future improvements such as authentication, HA Gateway, and deeper monitoring.

Current Architecture
Docker Nodes
Node	Hostname	IP Address	Swarm Role	Main Label
Main Host	reza-System-Product-Name	192.168.28.1	Manager / Leader	loki.role=main
Write Node	write	192.168.28.135	Manager	loki.role=write
Read Node	read	192.168.28.136	Manager	loki.role=read
Backend Node	other	192.168.28.137	Manager	loki.role=backend

Note: In this lab all four nodes are Docker Swarm managers. For production, an odd number of managers is recommended, typically 3 or 5.

Current Services
Main Host - 192.168.28.1
Service	Purpose	Port
Nginx Loki Gateway	Main Loki entrypoint for push and query traffic	3100
Grafana	UI for log visualization and querying	3000
MinIO	S3-compatible object storage for Loki	9000
MinIO Console	MinIO web UI	9001
Alloy	Docker log collector for the main host	12345

Write Node - 192.168.28.135
Service	Count	Port
Loki Distributor	3	3101, 3102, 3103
Loki Ingester	3	internal
WAL Volumes	3	one dedicated volume per Ingester

Read Node - 192.168.28.136
Service	Count	Port
Loki Query Frontend	2	3201, 3202
Loki Querier	3	internal

Backend Node - 192.168.28.137
Service	Count	Port
Consul	1	8500
Loki Query Scheduler	2	internal
Loki Index Gateway	2	internal
Loki Compactor	1	internal


Main Traffic Flows
Write / Ingestion Path
Docker container logs
        ↓
Alloy on main-host
        ↓
Nginx Loki Gateway - 192.168.28.1:3100
        ↓
Distributors on write node:
  192.168.28.135:3101
  192.168.28.135:3102
  192.168.28.135:3103
        ↓
Ingesters x3
        ↓
MinIO object storage - 192.168.28.1:9000

Read / Query Path
Grafana - 192.168.28.1:3000
        ↓
Nginx Loki Gateway - 192.168.28.1:3100
        ↓
Query Frontends on read node:
  192.168.28.136:3201
  192.168.28.136:3202
        ↓
Query Scheduler x2
        ↓
Queriers x3
        ↓
Index Gateway x2 / MinIO

Ring / KV Path
Loki components
        ↓
Consul - 192.168.28.137:8500

Consul stores Loki ring data, token ownership, internal state, and coordination metadata.

Project File Structure
Recommended repository structure:
loki-stack/
├── loki-stack-split.yml
├── configs/
│   ├── alloy-config.alloy
│   ├── loki-common-v1.yml
│   └── loki-gateway-split-v1.conf
├── docs/
│   └── architecture.md
└── README.md

Main Files
File	Description
loki-stack-split.yml	Main Docker Stack file for all services
configs/loki-common-v1.yml	Shared Loki configuration used by all Loki components
configs/loki-gateway-split-v1.conf	Nginx Gateway configuration for read/write routing
configs/alloy-config.alloy	Alloy configuration for Docker log collection
README.md	Main GitHub documentation
docs/architecture.md	Architecture notes and design details


Prerequisites
·	Docker Engine installed on all nodes.
·	Docker Swarm enabled.
·	Network connectivity between all nodes.
·	Required images available on the target nodes.
·	Docker node labels configured for service placement.
·	Required ports open between nodes.
Images Used
grafana/loki:2.9.10
grafana/grafana:latest
grafana/alloy:latest
nginx:latest
minio/minio:latest
docker.arvancloud.ir/consul:1.15.4


Docker Node Labels
Service placement is controlled using Docker node labels.
Example:
docker node update --label-add loki.role=main reza-System-Product-Name
docker node update --label-add loki.role=write write
docker node update --label-add loki.role=read read
docker node update --label-add loki.role=backend other

Check labels:
docker node inspect self --format '{{json .Spec.Labels}}'
docker node ls


Deploying the Stack
Run from any Docker Swarm manager node:
cd ~/loki-stack
docker stack deploy -c loki-stack-split.yml loki

Check service status:
docker service ls
docker stack ps loki


Stopping the Stack
To remove the stack services without deleting volumes or project files:
docker stack rm loki

Watch until services and containers are removed:
watch -n 2 'docker service ls; echo; docker ps'

This does not delete Docker volumes, MinIO data, or local configuration files.

Starting the Stack Again
cd ~/loki-stack
docker stack deploy -c loki-stack-split.yml loki
sleep 30
docker service ls
curl -s http://192.168.28.1:3100/ready


Health Checks and Useful Endpoints
Alloy
curl -s http://192.168.28.1:12345/-/ready
curl -s http://192.168.28.1:12345/-/healthy

Alloy UI:
http://192.168.28.1:12345

Loki Gateway
curl -s http://192.168.28.1:3100/ready

Distributors
curl -s http://192.168.28.135:3101/ready
curl -s http://192.168.28.135:3102/ready
curl -s http://192.168.28.135:3103/ready

Query Frontends
curl -s http://192.168.28.136:3201/ready
curl -s http://192.168.28.136:3202/ready

Consul
curl -s http://192.168.28.137:8500/v1/status/leader

MinIO Console
http://192.168.28.1:9001


Testing Loki Push
Manual log push example:
curl -v -X POST "http://192.168.28.1:3100/loki/api/v1/push" \
  -H "Content-Type: application/json" \
  --data-raw "{\"streams\":[{\"stream\":{\"job\":\"manual-test\",\"host\":\"gateway\"},\"values\":[[\"$(date +%s%N)\",\"hello from manual test $(date -u)\"]]}]}"


Testing Loki Query
curl -G -s "http://192.168.28.1:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={job="docker"}' \
  --data-urlencode 'limit=10'

Example query for the Gateway service:
{job="docker", service="loki-gateway"}

Example query for Grafana:
{job="docker", service="grafana"}

Example query for MinIO:
{job="docker", service="minio"}


Loki Label Design
Initially, the log streams included noisy and high-cardinality labels such as container_id. This made Grafana's label browser harder to use and increased label cardinality.
The current Alloy configuration keeps labels simple:
job
host
service

Example:
job="docker"
host="main-host"
service="loki-gateway"

Design goals:
·	Remove container_id.
·	Remove long and low-value Docker labels.
·	Make Grafana Explore cleaner.
·	Reduce cardinality.
·	Allow service-level filtering using service.

Grafana Label Browser Notes
If old labels such as container_id are still visible in Grafana, it usually means old log streams are still included in the selected time range.
To validate the new labels:
·	Set Grafana Explore time range to Last 5 minutes.
·	Refresh Explore.
·	Generate a new log entry, for example:
curl -s http://192.168.28.1:3100/ready >/dev/null

Then query:
{job="docker", service=~".+"}


Issues Solved During Implementation
query filtering for deletes Error
The read component initially failed with:
query filtering for deletes requires 'compactor_grpc_address' or 'compactor_address' to be configured

Resolution:
·	Added compactor_grpc_address.
·	Ensured query/read components can communicate with the compactor.
too many unhealthy instances in the ring
Queries initially failed with:
too many unhealthy instances in the ring

Resolution:
·	Corrected ring configuration.
·	Added proper ingester.lifecycler.ring settings.
·	Configured replication_factor.
·	Moved the ring backend from memberlist to Consul.
Consul Multiple Private IP Error
Consul initially failed with:
Multiple private IPv4 addresses found. Please configure one with 'bind' and/or 'advertise'.

Resolution:
·	Configured the correct bind/advertise address.
·	Ran Consul on the backend node with IP 192.168.28.137.
MinIO Connectivity Timeout
Some queries timed out when Loki components resolved minio:9000 to an internal overlay address.
Resolution:
·	Used the real MinIO endpoint in Loki configuration:
endpoint: 192.168.28.1:9000

Alloy Configuration Syntax Error
Alloy failed because the River config used # comments.
Example error:
illegal character U+0023 '#'

Resolution:
·	Removed invalid # comments from the Alloy configuration.
·	Used valid River syntax.
System Slowness
The environment became slow because Alloy was repeatedly failing and restarting, and the main host disk was almost full.
Resolution:
·	Temporarily scaled Alloy to zero.
·	Fixed the Alloy configuration.
·	Freed disk space.
·	Started Alloy again after validating the config.

Final Scale State
Component	Current Count
Distributor	3
Ingester	3
Querier	3
Query Frontend	2
Query Scheduler	2
Index Gateway	2
Compactor	1
Consul	1
MinIO	1
Nginx Gateway	1
Grafana	1
Alloy	1


Production Readiness Notes
This project is a production-oriented lab, but it is not yet a full production deployment.
Recommended production improvements:
Gateway High Availability
The Nginx Gateway is currently a single entrypoint:
192.168.28.1:3100

Production options:
·	Keepalived + VIP
·	HAProxy
·	External Load Balancer
·	Multiple Gateway replicas on different nodes
Consul High Availability
Consul is currently single-instance. For production, use at least a 3-node Consul cluster.
MinIO High Availability
MinIO is currently single-instance. For production, use MinIO distributed mode or an external object storage service.
Authentication
Loki currently runs without internal authentication. A practical next step is:
Grafana / Alloy -> Nginx Gateway with Basic Auth -> Loki

TLS
TLS should be enabled at the Gateway for production.
Monitoring
For deeper monitoring:
·	Scrape /metrics.
·	Add Prometheus.
·	Create Grafana dashboards for Loki, Docker, and Swarm.
·	Add alerts for unhealthy ring members, failed services, disk usage, and high query latency.

Useful Troubleshooting Commands
Service Status
docker service ls
docker stack ps loki

Service Logs
docker service logs --tail=100 loki_loki-querier
docker service logs --tail=100 loki_alloy

Ring Status
curl -s http://192.168.28.135:3101/ring | grep -E "ACTIVE|UNHEALTHY|JOINING|LEAVING|Forget"

Loki Labels
curl -G -s "http://192.168.28.1:3100/loki/api/v1/labels" | jq
curl -G -s "http://192.168.28.1:3100/loki/api/v1/label/service/values" | jq

Resource Usage
uptime
free -h
df -h
docker stats --no-stream

Docker Disk Usage
docker system df
sudo du -h -d1 /var/lib/docker | sort -h


Pushing the Project to GitHub
Yes, this project can be pushed to GitHub and maintained as a personal or internal repository.
Example:
cd ~/loki-stack

git init
git add .
git commit -m "Initial Loki Docker Swarm stack"

git branch -M main
git remote add origin git@github.com:<USERNAME>/<REPOSITORY>.git
git push -u origin main

Using HTTPS:
git remote add origin https://github.com/<USERNAME>/<REPOSITORY>.git
git push -u origin main

Recommended .gitignore
*.bak
*.backup
*.tmp
.env
secrets/
data/
minio-data/
grafana-data/

Do not commit credentials, tokens, private keys, or production secrets. If MinIO or Grafana credentials are currently inside the stack file, move them to Docker secrets or an environment file that is not committed.

Summary
This project implements a multi-component Loki logging stack on Docker Swarm. The current architecture includes separated write and read paths, MinIO object storage, Consul-based ring state, an Nginx Gateway, Grafana visualization, and Alloy-based Docker log collection.
The stack is suitable for a lab, learning Loki split deployment, architecture validation, and preparing for production. For a production-grade deployment, the next recommended improvements are Gateway HA, Consul HA, MinIO HA, authentication, TLS, and full metrics-based monitoring.
