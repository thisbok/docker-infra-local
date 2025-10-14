# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Docker-based ELK (Elasticsearch, Logstash, Kibana) stack with Redis and MySQL integration for log processing and analytics. The setup uses Docker Compose to orchestrate multiple services including Elasticsearch, Logstash, Kibana, Redis, and MySQL.

## Common Commands

### Initial Setup
```bash
# Initialize Elasticsearch users and passwords
docker-compose up setup

# Start all services
docker-compose up

# Start services in background
docker-compose up -d
```

### Service Management
```bash
# Restart specific services (e.g., after config changes)
docker-compose up -d logstash kibana

# View logs
docker-compose logs [service-name]

# Stop and remove all containers and data
docker-compose down -v

# Rebuild images after version changes
docker-compose build
```

### User Management
```bash
# Reset user passwords
docker-compose exec elasticsearch bin/elasticsearch-reset-password --batch --user elastic
docker-compose exec elasticsearch bin/elasticsearch-reset-password --batch --user logstash_internal
docker-compose exec elasticsearch bin/elasticsearch-reset-password --batch --user kibana_system
```

## Architecture

### Services and Ports
- **Elasticsearch**: 9200 (HTTP), 9300 (TCP transport)
- **Logstash**: 5044 (Beats input), 50000 (TCP input), 9600 (monitoring API)
- **Kibana**: 5601 (Web UI)
- **Redis**: 6379
- **MySQL**: 3306
- **Jaeger**: 16686 (Web UI), 14268 (HTTP collector), 14250 (gRPC), 6831/6832 (UDP), 5778 (configs)
- **Zookeeper**: 2181 (Kafka coordination)
- **Kafka**: 9092 (external), 29092 (internal), 9094 (JMX)
- **Kafka UI**: 8080 (Web UI)

### Data Flow
- **Redis**: Message queue receiving JSON data in lists
- **Kafka**: Streaming platform for logs, events, and metrics (topics: logs, events, metrics)
- **Logstash**:
  - Pulls data from Redis using the `transactions` key
  - Consumes data from Kafka topics
  - Processes and filters all data streams
- **Elasticsearch**: Central data store for all processed data
- **Kibana**: Visualization and analytics interface
- **Jaeger**: Distributed tracing visualization and analysis

### Key Configuration Files
- **Environment variables**: `.env` - Contains passwords and version settings
- **Docker services**: `docker-compose.yml` - Service definitions and networking
- **Logstash pipeline**: `logstash/pipeline/logstash.conf` - Data processing pipeline
- **Logstash config**: `logstash/config/logstash.yml` - Service configuration
- **Elasticsearch config**: `elasticsearch/config/elasticsearch.yml`
- **Kibana config**: `kibana/config/kibana.yml`
- **MySQL config**: `mysql/config/my.cnf` - MySQL server configuration

### Redis Integration
The Logstash pipeline is configured to:
- Connect to Redis service using hostname "redis"
- Read JSON-formatted data from the "transactions" list
- Process data through filters before sending to Elasticsearch
- Use dynamic index targeting via `[@metadata][target_index]` field

### Extensions
The `extensions/` directory contains optional integrations:
- Enterprise Search, Filebeat, Metricbeat, Heartbeat, Curator, Fleet, Logspout
- Each extension has its own configuration and documentation

### Security
- Uses built-in Elasticsearch users with configurable passwords in `.env`
- Setup service initializes users and roles on first run
- Password reset procedures available for production use

## Version Management
- Stack version controlled via `ELASTIC_VERSION` in `.env` file
- Current version: 8.11.1
- Rebuild required when changing versions: `docker-compose build`

### MySQL Integration
- **Database**: `elk_db` - Main database for application data
- **User**: `elk_user` - Application database user
- **Connection**: Available on port 3306 with credentials from `.env`
- **Configuration**: MySQL 8.0 with UTF-8 support and performance tuning

### MySQL Management
```bash
# Connect to MySQL
docker-compose exec mysql mysql -u elk_user -p elk_db

# View MySQL logs
docker-compose logs mysql

# Reset MySQL (remove data)
docker-compose down -v && docker volume rm docker-elk-redis_mysql
```

### Jaeger Integration
- **Distributed Tracing**: Jaeger UI available at http://localhost:16686
- **Storage**: Uses Elasticsearch as backend storage
- **Data Input**: Accepts traces via HTTP (port 14268), gRPC (port 14250), UDP (ports 6831/6832)
- **Index Pattern**: Trace data stored with `jaeger` prefix
- **Compatibility**: Supports OpenTelemetry and native Jaeger protocols

### Jaeger Management
```bash
# View Jaeger logs
docker-compose logs jaeger

# Access Jaeger UI
open http://localhost:16686

# Send trace data via HTTP
curl -X POST http://localhost:14268/api/traces \
  -H "Content-Type: application/json" \
  -d '{
    "data": [{
      "traceID": "1234567890abcdef",
      "spans": [{
        "traceID": "1234567890abcdef",
        "spanID": "abcdef1234567890",
        "operationName": "test-operation",
        "startTime": '$(date +%s%N | cut -b1-16)',
        "duration": 50000,
        "tags": [{
          "key": "http.method",
          "type": "string",
          "value": "GET"
        }],
        "process": {
          "serviceName": "my-service",
          "tags": []
        }
      }]
    }]
  }'
```

### Kafka Integration
- **Message Streaming**: Apache Kafka for high-throughput event streaming
- **Topics**: Preconfigured topics - `logs`, `events`, `metrics`
- **Consumer**: Logstash consumes from all configured topics
- **UI**: Kafka UI available at http://localhost:8080
- **Index Pattern**: Kafka data stored in daily indices `kafka-logs-YYYY.MM.dd`

### Kafka Management
```bash
# View Kafka logs
docker-compose logs kafka

# Access Kafka UI
open http://localhost:8080

# Create topic manually
docker-compose exec kafka kafka-topics --create \
  --bootstrap-server localhost:9092 \
  --topic new-topic \
  --partitions 3 \
  --replication-factor 1

# List topics
docker-compose exec kafka kafka-topics --list \
  --bootstrap-server localhost:9092

# Send test message
docker-compose exec kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic logs
# Type messages and press Enter

# Consume messages
docker-compose exec kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic logs \
  --from-beginning
```

### Kafka Data Examples
```bash
# Send log data to Kafka
echo '{"timestamp":"2023-12-01T10:00:00Z","level":"INFO","message":"Application started","service":"web-app"}' | \
docker-compose exec -T kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic logs

# Send event data to Kafka
echo '{"timestamp":"2023-12-01T10:00:00Z","event":"user_login","user_id":"123","ip":"192.168.1.1"}' | \
docker-compose exec -T kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic events

# Send metric data to Kafka
echo '{"timestamp":"2023-12-01T10:00:00Z","metric":"cpu_usage","value":75.5,"host":"server-01"}' | \
docker-compose exec -T kafka kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic metrics
```

## Development Notes
- Configuration changes require service restarts
- Memory allocation can be tuned via `ES_JAVA_OPTS` and `LS_JAVA_OPTS`
- Data persisted in Docker volumes by default
- Single-node Elasticsearch configuration for development
- MySQL uses volume persistence for data safety