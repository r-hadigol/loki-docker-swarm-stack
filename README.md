# Loki Docker Swarm Stack

This repository contains a Docker Swarm based Grafana Loki logging stack.

## Current Architecture

The stack runs on four Docker Swarm nodes:

| Node | IP | Purpose |
|---|---|---|
| main | 192.168.28.1 | Gateway, Grafana, MinIO, Alloy |
| write | 192.168.28.135 | Distributors and Ingesters |
| read | 192.168.28.136 | Query Frontends and Queriers |
| backend | 192.168.28.137 | Consul, Query Scheduler, Index Gateway, Compactor |

## Main Components

- Grafana Loki 2.9.10
- Grafana
- Grafana Alloy
- Nginx Gateway
- MinIO
- Consul
- Docker Swarm

## Traffic Flow

Write path:

```text
Alloy -> Nginx Gateway -> Distributors -> Ingesters -> MinIO
