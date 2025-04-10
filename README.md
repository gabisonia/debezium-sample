# Debezium Outbox Pattern with PostgreSQL, Kafka, and Docker

This cheat sheet helps you quickly set up Debezium CDC with PostgreSQL using Docker Compose and validate that messages are published to Kafka.

---

## 🐳 1. Start Docker Compose

```bash
docker-compose up -d
```

## To restart clean (removing volumes and data):

```bash
docker-compose down -v
docker-compose up -d
```

## Check Container Status

```bash
docker-compose ps
```

## Connect to PostgreSQL

```bash
docker exec -it pg-outbox psql -U postgres -d mydb
```

## Create Outbox Schema and Table

```bash
CREATE SCHEMA IF NOT EXISTS outbox;

CREATE TABLE outbox.events (
    id UUID PRIMARY KEY,
    aggregate_type TEXT NOT NULL,
    aggregate_id TEXT NOT NULL,
    type TEXT NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);
```

## Insert Test Events

First, enable UUID generation (optional):


```bash
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
```

```bash
INSERT INTO outbox.events (id, aggregate_type, aggregate_id, type, payload)
VALUES (gen_random_uuid(), 'order', 'order-123', 'OrderCreated', '{"total": 49.99}');
```

## Register the Debezium Connector (using curl)

```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "outbox-connector",
    "config": {
      "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
      "plugin.name": "pgoutput",
      "tasks.max": "1",
      "database.hostname": "postgres",
      "database.port": "5432",
      "database.user": "postgres",
      "database.password": "postgres",
      "database.dbname": "mydb",
      "topic.prefix": "outboxserver",
      "schema.include.list": "outbox",
      "table.include.list": "outbox.events",
      "transforms": "unwrap",
      "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
      "transforms.unwrap.drop.tombstones": "true"
    }
  }'
```

## View Published Kafka Messages (via Kafka UI)

- Open browser at: http://localhost:8080
- Click on topic: outboxserver.outbox.events
- View messages under the Messages tab

### View Kafka Messages via CLI
```bash
docker exec -it <kafka_container> \
  kafka-console-consumer \
  --bootstrap-server kafka:29092 \
  --topic outboxserver.outbox.events \
  --from-beginning

```

