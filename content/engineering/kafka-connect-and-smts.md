+++
title = "Kafka Connect and SMTs: The Swiss Army Knife You're Probably Underusing"
date = 2026-01-19
[taxonomies]
tags = ["kafka", "messaging", "integration"]
+++

Kafka Connect is one of those parts of the Kafka ecosystem that doesn't get enough credit. It's not flashy. It doesn't have a cool streaming DSL. But it solves the single most common Kafka integration problem - getting data in and out of external systems - with minimal custom code.

I've gone from writing bespoke producer/consumer applications for every database-to-Kafka pipeline to deploying a JSON config file and calling it a day. Let me tell you why.

## The Architecture

Kafka Connect runs as a cluster of worker nodes. You deploy connectors to the cluster via a REST API. Each connector is either a *source* (reads from an external system, writes to Kafka) or a *sink* (reads from Kafka, writes to an external system).

Workers distribute connector tasks across the cluster. If a worker dies, its tasks get rebalanced to surviving workers. The framework handles offset tracking, error handling, and retries. You configure the connector. Kafka Connect does the plumbing.

There are two deployment modes:

- **Standalone:** Single worker, good for development and testing. Config is a properties file.
- **Distributed:** Multiple workers coordinating via Kafka. Config is submitted via REST API. This is what you run in production.

```bash
# deploying a connector to distributed mode
curl -X POST http://connect-worker:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "postgres-source",
    "config": {
      "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
      "database.hostname": "postgres",
      "database.port": "5432",
      "database.user": "replicator",
      "database.password": "${vault:db-password}",
      "database.dbname": "orders",
      "topic.prefix": "cdc",
      "plugin.name": "pgoutput"
    }
  }'
```

## Running on Kubernetes

Kafka Connect on Kubernetes with Strimzi is genuinely pleasant. You define a `KafkaConnect` custom resource, and Strimzi handles the deployment, scaling, and connector plugin management.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  name: my-connect-cluster
  annotations:
    strimzi.io/use-connector-resources: "true"
spec:
  replicas: 3
  bootstrapServers: kafka-cluster:9093
  build:
    output:
      type: docker
      image: registry.example.com/kafka-connect:latest
    plugins:
      - name: debezium-postgres
        artifacts:
          - type: maven
            group: io.debezium
            artifact: debezium-connector-postgres
            version: 2.5.0
```

Strimzi builds a custom Docker image with your connector plugins baked in. No more downloading JARs and managing classpaths manually. The `KafkaConnector` CRD lets you manage connectors declaratively alongside the rest of your Kubernetes manifests.

## Single Message Transforms: The Swiss Army Knife

SMTs (Single Message Transforms) are lightweight transformations applied to each message as it flows through a connector. They run inside the Connect worker, before the message hits Kafka (for source connectors) or before it's written to the sink (for sink connectors).

They're not meant for heavy processing - that's what Kafka Streams or Flink are for. But for the 80% of cases where you just need to massage the data a little, SMTs eliminate the need for an intermediate stream processing application.

### Common SMTs I Use Constantly

**ExtractField:** Pull a single field out of a struct.

```json
"transforms": "extractPayload",
"transforms.extractPayload.type": "org.apache.kafka.connect.transforms.ExtractField$Value",
"transforms.extractPayload.field": "after"
```

This one is a staple for Debezium CDC sources. The raw Debezium envelope has `before`, `after`, `source`, `op` fields. Often your downstream consumers only care about the `after` value. Extract it and simplify their lives.

**TimestampRouter:** Route messages to different topics based on timestamp.

```json
"transforms": "routeByDate",
"transforms.routeByDate.type": "org.apache.kafka.connect.transforms.TimestampRouter",
"transforms.routeByDate.topic.format": "${topic}-${timestamp}",
"transforms.routeByDate.timestamp.format": "yyyyMMdd"
```

Great for time-partitioned sink topics. Each day's data goes to a different topic, making retention management trivial.

**ValueToKey:** Set the message key from a field in the value.

```json
"transforms": "keyFromValue",
"transforms.keyFromValue.type": "org.apache.kafka.connect.transforms.ValueToKey",
"transforms.keyFromValue.fields": "order_id"
```

CDC events often come with a null key or a composite key that's inconvenient for downstream consumers. This lets you set a clean business key.

**Filter (with predicates):** Conditionally drop messages.

```json
"transforms": "dropDeletes",
"transforms.dropDeletes.type": "org.apache.kafka.connect.transforms.Filter",
"transforms.dropDeletes.predicate": "isDelete",
"predicates": "isDelete",
"predicates.isDelete.type": "org.apache.kafka.connect.transforms.predicates.HeaderStringEquals",
"predicates.isDelete.name": "__op",
"predicates.isDelete.expected": "d"
```

### Chaining SMTs

You can chain multiple SMTs in order. They execute in the sequence you define them.

```json
"transforms": "extractAfter,setKey,dropTombstones",
"transforms.extractAfter.type": "org.apache.kafka.connect.transforms.ExtractField$Value",
"transforms.extractAfter.field": "after",
"transforms.setKey.type": "org.apache.kafka.connect.transforms.ValueToKey",
"transforms.setKey.fields": "id",
"transforms.dropTombstones.type": "org.apache.kafka.connect.transforms.Filter",
"transforms.dropTombstones.predicate": "isTombstone",
"predicates": "isTombstone",
"predicates.isTombstone.type": "org.apache.kafka.connect.transforms.predicates.RecordIsTombstone"
```

This chain takes a Debezium CDC event, extracts the `after` state, sets the key to the `id` field, and drops tombstone records. Three transforms, zero custom code.

## Testing Connectors

Testing Kafka Connect setups is something most teams skip, and it shows in production. Here's what works:

**Testcontainers for integration tests.** Spin up Kafka, Connect, and your source/sink database in containers. Deploy the connector, produce/consume test data, verify the result.

```java
@Testcontainers
class PostgresSourceConnectorTest {
    @Container
    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.0"));

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    // deploy connector, insert rows, verify Kafka messages
}
```

**Connector config validation.** Before deploying, use the Connect REST API's validation endpoint:

```bash
curl -X PUT http://connect:8083/connector-plugins/PostgresConnector/config/validate \
  -H "Content-Type: application/json" \
  -d @connector-config.json
```

This catches configuration errors before they hit the cluster.

**Monitor connector status.** A deployed connector can be in a RUNNING, PAUSED, or FAILED state. Monitor this. We've had connectors silently fail due to schema changes in the source database, and nobody noticed until downstream consumers started complaining about stale data.

## The Bottom Line

Kafka Connect isn't glamorous, but it's the right tool for data integration. SMTs handle the light transformations that would otherwise require a separate microservice. Running on Kubernetes with Strimzi makes operations manageable. And testing - actually testing your connector configurations before production - will save you from the kind of silent failures that erode trust in your data pipelines.

Use Kafka Connect for what it's good at, and save the custom code for problems that actually need custom code.
