# **ELK Stack Hands-On Training Guide**

**Goal:** Ingest Linux system logs → Process in Logstash → Store in Elasticsearch → Visualize in Kibana
**Duration:** \~2 Hours
**Audience:** DevOps Engineers / Learners

---

# **1. Installation**

We'll use **Docker Compose** for a quick setup so that learners don't struggle with manual package installations.

### **Step 1.1 — Install Docker & Docker Compose**

For **Ubuntu / Debian**:

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-plugin
sudo systemctl enable docker --now
```

Verify:

```bash
docker --version
docker compose version
```

---

### **Step 1.2 — Create Project Structure**

```bash
mkdir elk-lab && cd elk-lab
mkdir -p logstash/pipeline filebeat
```

---

### **Step 1.3 — Create `docker-compose.yml`**

```yaml
version: "3.8"

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.2
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    ports:
      - "9200:9200"
    volumes:
      - esdata:/usr/share/elasticsearch/data

  kibana:
    image: docker.elastic.co/kibana/kibana:8.13.2
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch
    ports:
      - "5601:5601"

  logstash:
    image: docker.elastic.co/logstash/logstash:8.13.2
    container_name: logstash
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    ports:
      - "5044:5044"
      - "9600:9600"
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.13.2
    container_name: filebeat
    user: root
    command: ["--strict.perms=false"]
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/log:/hostfs/var/log:ro
    depends_on:
      - logstash

volumes:
  esdata:
```

---

### **Step 1.4 — Start the ELK Stack**

```bash
docker compose up -d
docker compose ps
```

Verify Elasticsearch:

```bash
curl -s http://localhost:9200 | jq .
```

Expected output:

```json
{
  "name" : "elasticsearch",
  "cluster_name" : "docker-cluster",
  "status" : "green",
  "tagline" : "You Know, for Search"
}
```

---

# **2. Configuration**

Now that the ELK stack is running, we’ll configure Logstash, Filebeat, and Kibana.

---

### **Step 2.1 — Configure Logstash**

Create `logstash/pipeline/logstash.conf`:

```conf
input {
  beats {
    port => 5044
  }
}

filter {
  grok {
    match => {
      "message" => "%{SYSLOGTIMESTAMP:timestamp} %{HOSTNAME:host} %{DATA:process}(?:\\[%{POSINT:pid}\\])?: %{GREEDYDATA:log_message}"
    }
  }

  date {
    match => ["timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss"]
    target => "@timestamp"
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "linux-logs-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}
```

Restart Logstash:

```bash
docker compose restart logstash
docker logs -f logstash
```

---

### **Step 2.2 — Configure Filebeat**

Create `filebeat/filebeat.yml`:

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /hostfs/var/log/syslog
      - /hostfs/var/log/auth.log
      - /hostfs/var/log/messages
    fields:
      source_type: system
    fields_under_root: true

processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~

output.logstash:
  hosts: ["logstash:5044"]
```

Restart Filebeat:

```bash
docker compose restart filebeat
docker logs -f filebeat
```

---

# **3. Setup & Demo**

### **Step 3.1 — Generate Test Logs**

```bash
logger "ELK_DEMO: user logged in successfully"
logger "ELK_DEMO: failed SSH login attempt from 192.168.1.100"
```

---

### **Step 3.2 — Verify Logs in Elasticsearch**

Check indices:

```bash
curl -s 'http://localhost:9200/_cat/indices?v'
```

Search for demo logs:

```bash
curl -s "http://localhost:9200/linux-logs-*/_search?q=ELK_DEMO&pretty"
```

---

### **Step 3.3 — Configure Kibana**

1. Open **Kibana** → [http://localhost:5601](http://localhost:5601).
2. Go to **Stack Management → Data Views → Create Data View**.
3. Name: `linux-logs`.
4. Index pattern: `linux-logs-*`.
5. Time field: `@timestamp`.

---

### **Step 3.4 — Explore Logs in Kibana**

* Go to **Discover**.
* Search: `ELK_DEMO`.
* Verify your logs appear.

---

### **Step 3.5 — Create Visualizations**

#### **Visualization 1 — Pie Chart for Processes**

* Go to **Visualize Library → Create Visualization → Pie Chart**.
* Buckets → Split Slices → `process.keyword`.
* Metric → Count.

#### **Visualization 2 — Failed SSH Login Trend**

* Go to **Visualize → Create → Line Chart**.
* X-Axis → `@timestamp`.
* Y-Axis → Count.
* Add filter: `log_message: "failed"`.

#### **Add to Dashboard**

* Go to **Dashboard → Create Dashboard**.
* Add both visualizations.

---

# **4. Verification**

| **Component** | **Command**                          | **Expected Result**     |
| ------------- | ------------------------------------ | ----------------------- |
| Elasticsearch | `curl localhost:9200`                | Status = green          |
| Logstash      | `docker logs logstash`               | Events received         |
| Filebeat      | `docker logs filebeat`               | Events sent to Logstash |
| Indices       | `curl localhost:9200/_cat/indices?v` | `linux-logs-*` present  |
| Kibana        | Open Discover                        | Logs visible            |
| Dashboard     | Open Dashboard                       | Charts populated        |

---

# **5. Troubleshooting**

| **Issue**                | **Cause**                 | **Fix**                        |
| ------------------------ | ------------------------- | ------------------------------ |
| Kibana shows no logs     | Wrong index pattern       | Use `linux-logs-*`             |
| Filebeat not starting    | Permissions on `/var/log` | Run Filebeat as `root`         |
| Logstash high CPU usage  | Complex Grok regex        | Simplify patterns              |
| No data in Elasticsearch | Wrong Logstash config     | Check `logstash.conf` pipeline |

---



Do you want me to prepare a **PDF version** of this guide with **screenshots** for each step — including Kibana dashboards — so that learners can follow visually during the session? It’ll make the 2-hour demo much smoother. Should I?
