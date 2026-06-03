# ELK Stack — Interview Fundamentals

## What is the ELK Stack?
**ELK** = **Elasticsearch** + **Logstash** + **Kibana**. It is an open-source platform for centralized log management, search, and analytics. Often extended to the **Elastic Stack** with **Beats** (lightweight shippers) and **APM**.

---

## Architecture Overview

```
Data Sources
  ├── Applications
  ├── Servers / VMs
  ├── Containers / Kubernetes
  └── Network devices
        │
        ▼
  [Beats / Filebeat / Metricbeat / Packetbeat]
        │  lightweight shippers
        ▼
  [Logstash]  ──── optional enrichment / transformation
        │  (or direct to ES via Beats)
        ▼
  [Elasticsearch]  ──── store, index, search
        │
        ▼
  [Kibana]  ──── visualize, dashboards, alerts
```

### Modern Ingest Options
| Method | Use Case |
|--------|---------|
| Filebeat → Elasticsearch | Simple log forwarding |
| Filebeat → Logstash → Elasticsearch | Parsing/enrichment needed |
| Beats → Kafka → Logstash → ES | High-volume, buffered pipeline |
| Fleet + Elastic Agent | Centrally managed, policy-based |

---

## Elasticsearch

### Core Concepts
| Concept | Description | SQL Analogy |
|---------|-------------|-------------|
| **Index** | Collection of documents | Table |
| **Document** | JSON record | Row |
| **Field** | Key-value pair in a document | Column |
| **Shard** | Horizontal subdivision of an index | Partition |
| **Replica** | Copy of a shard for HA and read scale | Replica |
| **Mapping** | Schema definition for an index | DDL |
| **Cluster** | One or more nodes holding all data | Database server |
| **Node** | Single Elasticsearch instance | |

### Node Roles
| Role | Purpose |
|------|---------|
| Master | Cluster management (index creation, shard allocation) |
| Data | Store shards, execute queries |
| Ingest | Pre-process documents (ingest pipelines) |
| Coordinating | Route requests, merge results |
| ML | Machine learning jobs |

### Index & Sharding
```
Primary shards = set at creation (cannot change without reindex)
Replica shards = can change at any time

Recommendation:
- Primary shard size: 10GB–50GB
- Number of primaries × (1 + replicas) = total shards
- Avoid "shard explosion" — too many small shards degrade performance
```

```bash
# Create index
PUT /app-logs-2025-06
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "index.refresh_interval": "5s"
  },
  "mappings": {
    "properties": {
      "@timestamp": { "type": "date" },
      "level":      { "type": "keyword" },
      "message":    { "type": "text" },
      "service":    { "type": "keyword" },
      "host":       { "type": "keyword" },
      "duration_ms":{ "type": "long" }
    }
  }
}
```

### Field Types
| Type | Use Case |
|------|---------|
| `text` | Full-text search (analyzed) |
| `keyword` | Exact match, aggregations, sorting |
| `date` | Date/time values |
| `long`, `integer`, `float` | Numbers |
| `boolean` | True/false |
| `ip` | IP address |
| `geo_point` | Latitude/longitude |
| `nested` | Array of objects with relationships |

> Use `keyword` for log levels, service names, IDs. Use `text` for log messages.

---

## Elasticsearch Query DSL

### Basic Search
```json
GET /app-logs-*/_search
{
  "query": {
    "match": {
      "message": "connection refused"
    }
  },
  "size": 20,
  "from": 0,
  "sort": [
    { "@timestamp": { "order": "desc" } }
  ]
}
```

### Common Query Types
```json
// Term query — exact match (keyword fields)
{ "term": { "level": "ERROR" } }

// Terms query — multiple exact values
{ "terms": { "service": ["trading", "pricing"] } }

// Range query
{ "range": { "@timestamp": { "gte": "now-1h", "lte": "now" } } }
{ "range": { "duration_ms": { "gte": 1000 } } }

// Match query — full-text search (analyzed)
{ "match": { "message": "timeout error" } }

// Match phrase — exact phrase
{ "match_phrase": { "message": "connection timed out" } }

// Wildcard / prefix
{ "wildcard": { "host": "web-*" } }
{ "prefix": { "service": "trade" } }

// Exists
{ "exists": { "field": "error.stack_trace" } }
```

### Boolean Query (Most Common)
```json
GET /app-logs-*/_search
{
  "query": {
    "bool": {
      "must": [
        { "term":  { "level": "ERROR" } },
        { "range": { "@timestamp": { "gte": "now-24h" } } }
      ],
      "should": [
        { "match": { "message": "timeout" } },
        { "match": { "message": "refused" } }
      ],
      "must_not": [
        { "term": { "service": "health-check" } }
      ],
      "filter": [
        { "term": { "environment": "production" } }
      ]
    }
  }
}
```

> `must` → scored + filtered; `filter` → filtered only (cached, faster); `should` → at least one if no `must`/`filter`

### Aggregations
```json
GET /app-logs-*/_search
{
  "size": 0,
  "query": { "range": { "@timestamp": { "gte": "now-1d" } } },
  "aggs": {
    "errors_by_service": {
      "terms": {
        "field": "service",
        "size": 10
      },
      "aggs": {
        "error_count": {
          "filter": { "term": { "level": "ERROR" } }
        },
        "avg_duration": {
          "avg": { "field": "duration_ms" }
        }
      }
    },
    "errors_over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "calendar_interval": "1h"
      }
    }
  }
}
```

### Index Management
```bash
# Index aliases (zero-downtime reindex)
POST /_aliases
{
  "actions": [
    { "add": { "index": "app-logs-v2", "alias": "app-logs" } },
    { "remove": { "index": "app-logs-v1", "alias": "app-logs" } }
  ]
}

# Index templates (auto-apply settings to new indices)
PUT /_index_template/app-logs-template
{
  "index_patterns": ["app-logs-*"],
  "template": {
    "settings": { "number_of_shards": 3 },
    "mappings": { ... }
  }
}

# ILM (Index Lifecycle Management)
PUT /_ilm/policy/app-logs-policy
{
  "policy": {
    "phases": {
      "hot":    { "actions": { "rollover": { "max_size": "50gb", "max_age": "1d" } } },
      "warm":   { "min_age": "7d",  "actions": { "shrink": { "number_of_shards": 1 } } },
      "cold":   { "min_age": "30d", "actions": { "freeze": {} } },
      "delete": { "min_age": "90d", "actions": { "delete": {} } }
    }
  }
}
```

---

## Logstash

### Pipeline Structure
```
Input → Filter → Output
```

### Example Pipeline (`/etc/logstash/conf.d/app.conf`)
```ruby
input {
  beats {
    port => 5044
    ssl  => true
    ssl_certificate => "/etc/logstash/certs/logstash.crt"
    ssl_key         => "/etc/logstash/certs/logstash.key"
  }
  kafka {
    bootstrap_servers => "kafka:9092"
    topics            => ["app-logs"]
    group_id          => "logstash"
    codec             => "json"
  }
}

filter {
  # Parse JSON log
  json {
    source => "message"
    target => "parsed"
  }

  # Parse unstructured log with Grok
  grok {
    match => {
      "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} \[%{DATA:thread}\] %{GREEDYDATA:log_message}"
    }
    tag_on_failure => ["_grokparsefailure"]
  }

  # Parse timestamp
  date {
    match => ["timestamp", "ISO8601"]
    target => "@timestamp"
    remove_field => ["timestamp"]
  }

  # Enrich with GeoIP
  geoip {
    source => "client_ip"
    target => "geoip"
  }

  # Conditional logic
  if [level] == "ERROR" {
    mutate {
      add_tag => ["alert"]
      add_field => { "notify" => "true" }
    }
  }

  # Remove fields
  mutate {
    remove_field => ["agent", "ecs", "input"]
  }
}

output {
  elasticsearch {
    hosts           => ["https://es01:9200", "https://es02:9200"]
    index           => "app-logs-%{+YYYY.MM.dd}"
    user            => "${ES_USER}"
    password        => "${ES_PASS}"
    ssl_certificate_verification => true
    cacert          => "/etc/logstash/certs/ca.crt"
  }

  if "alert" in [tags] {
    http {
      url    => "http://alertmanager:9093/api/v1/alerts"
      http_method => "post"
      format => "json"
    }
  }
}
```

---

## Filebeat

### Configuration (`filebeat.yml`)
```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/myapp/*.log
      - /var/log/myapp/archive/*.log.gz
    multiline:
      pattern: '^\d{4}-\d{2}-\d{2}'  # new log starts with date
      negate: true
      match: after
    fields:
      service: myapp
      environment: production
    fields_under_root: true

  - type: container
    paths:
      - /var/lib/docker/containers/*/*.log
    processors:
      - add_docker_metadata: ~

processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~

output.logstash:
  hosts: ["logstash:5044"]
  ssl.certificate_authorities: ["/etc/filebeat/ca.crt"]

# OR direct to Elasticsearch
output.elasticsearch:
  hosts: ["https://es01:9200"]
  username: "${ES_USER}"
  password: "${ES_PASS}"
  index: "filebeat-%{[agent.version]}-%{+yyyy.MM.dd}"

monitoring.enabled: true
```

---

## Kibana

### Key Features
| Feature | Description |
|---------|-------------|
| **Discover** | Ad-hoc log search and exploration |
| **Lens / Visualize** | Build charts (bar, line, pie, heatmap) |
| **Dashboard** | Combine visualizations |
| **Maps** | Geo data visualization |
| **Alerting** | Rules-based alerts on index data |
| **APM** | Application performance monitoring |
| **Machine Learning** | Anomaly detection |
| **Canvas** | Pixel-perfect reports |

### Kibana Query Language (KQL)
```
level: ERROR
service: "trading-engine" and level: ERROR
message: "connection refused" and duration_ms > 1000
host: web-* and NOT service: health-check
@timestamp >= "2025-06-01" and @timestamp < "2025-06-02"
```

---

## Cluster Health & Operations

```bash
# Cluster health
GET /_cluster/health?pretty
GET /_cluster/health?wait_for_status=green&timeout=30s

# Node info
GET /_nodes?pretty
GET /_cat/nodes?v

# Index stats
GET /_cat/indices?v&s=store.size:desc
GET /_cat/shards?v

# Pending tasks
GET /_cluster/pending_tasks

# Force shard allocation
POST /_cluster/reroute
{
  "commands": [{
    "allocate_stale_primary": {
      "index": "app-logs-2025-06-01",
      "shard": 0,
      "node": "es-node-2",
      "accept_data_loss": false
    }
  }]
}

# Reindex
POST /_reindex
{
  "source": { "index": "app-logs-v1" },
  "dest":   { "index": "app-logs-v2" }
}
```

### Health Status
| Status | Meaning |
|--------|---------|
| 🟢 Green | All primary and replica shards assigned |
| 🟡 Yellow | All primaries assigned; some replicas unassigned |
| 🔴 Red | Some primary shards unassigned — data unavailable |

---

## Common Interview Questions

### Q1: What is an inverted index?
The data structure Elasticsearch uses for full-text search. It maps each unique term to the documents containing it, enabling fast lookups. The opposite of a forward index (document → words).

### Q2: What is the difference between `text` and `keyword`?
- `text`: analyzed (tokenized, lowercased, stop words removed) → full-text search
- `keyword`: not analyzed → exact match, aggregations, sorting, scripting

### Q3: What happens when you search with `must` vs `filter`?
- `must`: affects relevance scoring; query is cached less aggressively
- `filter`: no scoring; aggressively cached by ES; much faster for exact matches

### Q4: How does Elasticsearch scale?
Horizontally by adding nodes. Shards are automatically redistributed. Read scales with replicas (any shard copy can serve queries). Write scales with more primaries.

### Q5: What is an ILM policy?
Index Lifecycle Management automatically transitions indices through phases (hot/warm/cold/delete) based on age or size. Reduces storage costs and maintains query performance.

### Q6: What causes a yellow/red cluster?
- **Yellow**: Single-node cluster (no room for replicas), or node is down
- **Red**: Primary shard(s) unassigned — data loss risk or unreachable shard

### Q7: What is the difference between Logstash and Filebeat?
- **Filebeat**: Lightweight log shipper, low CPU/memory, runs on every host
- **Logstash**: Heavy-duty pipeline with rich filtering/transformation; centralized

### Q8: How do you handle multiline logs (e.g., Java stack traces)?
Use `multiline` processor in Filebeat or `multiline` codec in Logstash to group lines belonging to the same event based on a pattern (e.g., lines not starting with a date are continuation of previous event).

### Q9: What is a Grok pattern?
A named regex in Logstash/Elasticsearch used to parse unstructured text into structured fields. Built-in patterns cover common formats (TIMESTAMP, IP, LOGLEVEL, etc.).

### Q10: How do you secure the ELK stack?
- Enable TLS for inter-node and client communications
- Enable X-Pack security (Elastic Security, built-in from 7.x+)
- Role-based access control (RBAC) for index-level permissions
- Field- and document-level security for sensitive data
- Use secrets management (Keystore) for credentials
