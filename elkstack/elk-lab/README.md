
# ELK Stack Hands-On Lab (Beginner Friendly)

Ingest Linux system logs with **Filebeat**, process them with **Logstash**, store them in **Elasticsearch**, and visualize them in **Kibana** — all using **Docker Compose**.


## 🧭 Table of Contents

1. [What You’ll Build](#what-youll-build)
2. [Prerequisites](#prerequisites)
3. [Quick Start (Recommended)](#quick-start-recommended)
4. [Project Structure](#project-structure)
5. [Spin Up the Stack](#spin-up-the-stack)
6. [Generate Test Logs](#generate-test-logs)
7. [Verify Data End-to-End](#verify-data-end-to-end)
8. [Use Kibana (Discover, KQL, Visualizations, Dashboard)](#use-kibana-discover-kql-visualizations-dashboard)
9. [Troubleshooting Guide](#troubleshooting-guide)
10. [FAQ](#faq)
11. [Reset / Cleanup](#reset--cleanup)
12. [Next Steps](#next-steps)

---

## What You’ll Build

* A single-node **Elasticsearch** + **Kibana** + **Logstash** + **Filebeat** stack.
* Filebeat reads Linux system logs from your host and ships them to Logstash.
* Logstash parses syslog lines and indexes documents into Elasticsearch as `linux-logs-YYYY.MM.dd`.
* You’ll search, filter with **KQL**, and build visualizations in **Kibana**.

**Architecture:**
**Filebeat → Logstash → Elasticsearch → Kibana**

---

## Prerequisites

* A Linux machine (VM or bare metal) with **4+ GB RAM** and **2+ vCPUs**.
* **Docker** >= 24 and **Docker Compose** >= 2.
* Open local ports: `9200` (Elasticsearch), `5601` (Kibana), `5044` (Logstash).

### Install Docker (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-plugin
sudo systemctl enable docker --now
docker --version
docker compose version
```

---

## Quick Start (Recommended)

1. **Clone or download** this repository.
2. From the project folder, run:

   ```bash
   docker compose up -d
   docker compose ps
   ```
3. Confirm Elasticsearch is up:

   ```bash
   curl -s http://localhost:9200 | jq .
   ```
4. Generate test logs → then verify in Kibana.

---

## Project Structure

```
elk-lab/
├── docker-compose.yml          # Docker services for Elasticsearch, Kibana, Logstash, Filebeat
├── README.md                   # This guide
├── logstash/
│   └── pipeline/
│       └── logstash.conf       # Parses syslog and indexes into linux-logs-YYYY.MM.dd
└── filebeat/
    └── filebeat.yml            # Reads host logs (/var/log) and ships to Logstash
```

---

## Spin Up the Stack

From the `elk-lab` directory:

### 1) Start containers

```bash
docker compose up -d
docker compose ps
```

### 2) Verify each component

```bash
# Elasticsearch (expect JSON with cluster info)
curl -s http://localhost:9200 | jq .

# Kibana UI → http://localhost:5601 (may take 30-60s to start)

# Logstash logs
docker logs -f logstash

# Filebeat logs
docker logs -f filebeat
```

> **Tip:** If Filebeat fails with permission errors, we already run it as **root** and mount `/var/log` read-only. Check your distro’s log paths if needed.

---

## Generate Test Logs

On the **Linux host**:

```bash
logger "ELK_DEMO: user logged in successfully"
logger "ELK_DEMO: failed SSH login attempt from 192.168.1.100"
```

* **Ubuntu/Debian** → Logs in `/var/log/syslog` and `/var/log/auth.log`
* **RHEL/CentOS** → Logs in `/var/log/messages` and `/var/log/secure`

---

## Verify Data End-to-End

### 1) Check indices in Elasticsearch

```bash
curl -s 'http://localhost:9200/_cat/indices?v'
```

Expect something like:

```
yellow open linux-logs-2025.08.25 ...
```

### 2) Search for your test messages

```bash
curl -s "http://localhost:9200/linux-logs-*/_search?q=ELK_DEMO&pretty"
```

---

## Use Kibana (Discover, KQL, Visualizations, Dashboard)

### A) Open Kibana

* URL: **[http://localhost:5601](http://localhost:5601)**
* Wait \~30–60 seconds on first start.

### B) Create a Data View

1. Go to **Stack Management → Data Views → Create data view**.
2. **Name**: `linux-logs`
3. **Index pattern**: `linux-logs-*`
4. **Time field**: `@timestamp`
5. Click **Create data view**.

### C) Explore Logs in Discover

* Click **Discover** (left nav).
* Set **Last 15 minutes** in the time picker.
* Search for our test logs:

  ```
  message : "ELK_DEMO"
  ```
* Filter failed attempts:

  ```
  message : "failed"
  ```
* Filter by host:

  ```
  host.name : "YOUR_HOSTNAME"
  ```

### D) Create Visualizations

#### Pie Chart (Process Distribution)

1. Go to **Analytics → Visualize Library → Create Visualization**.
2. Choose **Pie** chart.
3. **Metric**: Count.
4. **Buckets → Split slices by** → `process.name.keyword`.
5. Save as **Process distribution**.

#### Line Chart (Failed SSH Attempts Over Time)

1. Create another visualization → **Line**.
2. **Metric**: Count.
3. **X-axis**: `@timestamp`.
4. Add a **Panel Filter**:

   ```
   message : "failed"
   ```
5. Save as **Failed SSH Attempts Over Time**.

### E) Build a Dashboard

1. Go to **Analytics → Dashboard → Create dashboard**.
2. Click **Add from library** and add the two visualizations.
3. Save as **System Logs Overview**.

---

## Troubleshooting Guide

| Problem                   | Cause                       | Solution                              |
| ------------------------- | --------------------------- | ------------------------------------- |
| No index created          | Filebeat → Logstash issue   | Check logs for both containers        |
| Filebeat permission error | `/var/log` mount restricted | Run Filebeat as root (already set)    |
| `_grok_parse_failure`     | Logstash parsing failed     | Comment out filter in `logstash.conf` |
| Kibana empty              | Wrong time filter           | Set "Last 15 minutes"                 |
| Direct test bypass        | Filebeat → Elasticsearch    | Configure output.elasticsearch        |

---

## FAQ

**Q: Where are configs?**

* `logstash/pipeline/logstash.conf`
* `filebeat/filebeat.yml`
* `docker-compose.yml`

**Q: Can I run on Windows/Mac?**
Yes, but Filebeat won’t read host logs directly — use container logs instead.

---

## Reset / Cleanup

Stop and remove containers + volumes (clears data):

```bash
docker compose down -v
```

---

## Next Steps

* Enable Filebeat system module dashboards:

  ```yaml
  setup.kibana.host: "http://kibana:5601"
  setup.dashboards.enabled: true
  ```
* Add Kafka or Redis for buffering.
* Secure ELK with TLS and authentication.

---

Do you also want me to create a **visual training flow** — showing **how learners should navigate Kibana step by step with screenshots**?
It’ll make the training more effective. Should I?
