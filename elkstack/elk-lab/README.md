# ELK Stack Hands-On Lab (Linux Log Ingestion)

This repository provides a ready-to-use ELK Stack lab for DevOps engineers. It demonstrates ingesting Linux system logs using Filebeat, parsing them with Logstash, storing them in Elasticsearch, and visualizing them via Kibana.

## Project Structure
```
elk-lab/
├── docker-compose.yml
├── README.md
├── logstash/
│   └── pipeline/
│       └── logstash.conf
└── filebeat/
    └── filebeat.yml
```

## Prerequisites
- Docker >= 24.x
- Docker Compose >= 2.x
- Minimum 4GB RAM, 2 vCPUs

## Setup
```bash
git clone <repo-url>
cd elk-lab
docker compose up -d
```

## Verify Elasticsearch
```bash
curl -s http://localhost:9200 | jq .
```

## Generate Test Logs
```bash
logger "ELK_DEMO: user logged in successfully"
logger "ELK_DEMO: failed SSH login attempt from 192.168.1.100"
```

## Check Indices
```bash
curl -s 'http://localhost:9200/_cat/indices?v'
```

## Kibana Setup
- Go to: http://localhost:5601
- Create Data View: `linux-logs-*`
- Visualize logs and create dashboards.
