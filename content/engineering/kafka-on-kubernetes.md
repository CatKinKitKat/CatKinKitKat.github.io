+++
title = "Running Kafka on Kubernetes: Because Apparently We Enjoy Complexity"
date = 2026-01-30
[taxonomies]
tags = ["kafka", "kubernetes", "operations"]
+++

Running a stateful distributed system on top of another distributed system sounds like the punchline to a bad joke. And yet, here we are. Kafka on Kubernetes is increasingly the default deployment model, and when done right, it actually works pretty well.

When done wrong, it's a masterclass in cascading failures. Let me share what I've learned from both outcomes.

## Strimzi: The Operator That Makes It Possible

Don't try to run Kafka on Kubernetes with raw StatefulSets and PersistentVolumeClaims. You'll lose months of your life to YAML hell. Use an operator. Strimzi is the open-source standard, and it's excellent.

Strimzi manages the entire Kafka lifecycle through Custom Resource Definitions: `Kafka`, `KafkaTopic`, `KafkaUser`, `KafkaConnect`, `KafkaMirrorMaker2`, and more. You declare your desired state, and the operator reconciles reality.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: production-cluster
spec:
  kafka:
    version: 3.7.0
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
    storage:
      type: persistent-claim
      size: 500Gi
      class: fast-ssd
    resources:
      requests:
        memory: 8Gi
        cpu: "2"
      limits:
        memory: 8Gi
        cpu: "4"
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 50Gi
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

The operator handles rolling upgrades, certificate management (if using TLS), topic management, and user/ACL management. It's not perfect - I have grievances about how it handles some edge cases during rolling restarts - but it's dramatically better than managing Kafka on Kubernetes manually.

## The Storage Problem

Kafka is a disk-heavy workload. Each broker can consume terabytes of storage depending on your retention policies. On Kubernetes, that means PersistentVolumeClaims backed by cloud block storage.

The critical decision: **storage class**. Use SSDs. Not "premium" HDDs, not "balanced" storage, SSDs. Kafka's performance is dominated by sequential disk I/O, and the difference between provisioned SSD IOPS and "general purpose" block storage is the difference between 100MB/s throughput and 20MB/s.

On Azure, that means `managed-premium` or `ultra-disk`. On AWS, `gp3` with provisioned IOPS at minimum, `io2` if you're serious.

Watch out for **PVC resizing**. Not all storage classes support volume expansion. If your broker runs out of disk and you can't expand the PVC, your options are ugly: add a new broker and rebalance partitions, or delete the broker pod and recreate it with a larger PVC (losing the local data, which then has to be replicated from other brokers). Plan your storage generously from the start.

## Autoscaling with KEDA

KEDA (Kubernetes Event-Driven Autoscaler) can scale your Kafka consumer deployments based on consumer group lag. More lag, more pods. Less lag, fewer pods.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 2
  maxReplicaCount: 12
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka-cluster:9092
        consumerGroup: order-processor-group
        topic: orders
        lagThreshold: "100"
        offsetResetPolicy: earliest
```

When the lag on the `orders` topic exceeds 100 messages per partition, KEDA scales up the consumer deployment. When lag drops, it scales back down.

Sounds great. In practice, there are gotchas.

### The Rebalancing Storm

Scaling up means new consumers join the group, which triggers a rebalance. During the rebalance, *all* consumers pause. If you scale up aggressively (say, from 3 to 10 pods in one shot), you get a rebalance pause that might be longer than the time savings from having more consumers.

Mitigations:
- Use `CooperativeStickyAssignor` to minimize rebalance impact
- Configure KEDA's `cooldownPeriod` and `pollingInterval` to avoid rapid scaling
- Set `stabilizationWindowSeconds` on the HPA behavior to smooth out scaling decisions

### Partition Ceiling

Remember: your maximum consumer parallelism is bounded by partition count. If your topic has 12 partitions, KEDA can scale to 12 pods max. Scaling to 15 just means 3 idle pods. Set `maxReplicaCount` to your partition count.

## Queue Depth Monitoring

Beyond KEDA's lag-based scaling, you need proper queue depth monitoring for operational visibility. The combination I've found most effective:

**Prometheus + kafka_exporter:** Exports per-partition, per-consumer-group lag metrics.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-exporter
spec:
  template:
    spec:
      containers:
        - name: kafka-exporter
          image: danielqsj/kafka-exporter:latest
          args:
            - --kafka.server=kafka-cluster:9092
            - --topic.filter=.*
            - --group.filter=.*
```

**Grafana dashboards** with alerts on:
- Consumer lag exceeding threshold (per partition, not just aggregate)
- Consumer group state changes (rebalancing, dead, empty)
- Lag growth rate (is lag increasing or just elevated but stable?)

The lag growth rate is the metric most teams miss. Lag of 10,000 that's been stable for an hour is very different from lag of 10,000 that was 5,000 an hour ago.

## Building Resilient Kafka Clients

Running on Kubernetes means your Kafka clients will deal with broker pod restarts during rolling updates. This is normal. Your clients need to handle it gracefully.

### Producer Resilience

```java
props.put(ProducerConfig.ACKS_CONFIG, "all");
props.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
props.put(ProducerConfig.RETRY_BACKOFF_MS_CONFIG, 100);
props.put(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, 120000);
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
```

With idempotent producers and retries, transient broker unavailability during a rolling restart is handled transparently. The producer retries sends until the broker comes back or the delivery timeout is exceeded.

### Consumer Resilience

```java
props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 30000);
props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 10000);
props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000);
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
```

The session timeout needs to be long enough to survive a brief broker restart without triggering a rebalance. Too short and every rolling restart triggers a rebalance cascade. Too long and genuinely dead consumers aren't detected promptly.

### Liveness and Readiness Probes

Don't use Kafka connectivity as a readiness probe. If the broker is briefly unreachable during a rolling update, your consumer pod fails its readiness check, gets pulled from the service, and potentially restarted - making everything worse.

Instead, use a simple health check that verifies the consumer thread is alive and the last poll was recent:

```java
@Component
public class KafkaHealthIndicator implements HealthIndicator {
    private volatile Instant lastPollTime = Instant.now();

    public void recordPoll() {
        this.lastPollTime = Instant.now();
    }

    @Override
    public Health health() {
        if (Duration.between(lastPollTime, Instant.now()).toSeconds() > 60) {
            return Health.down().withDetail("reason", "No poll in 60 seconds").build();
        }
        return Health.up().build();
    }
}
```

## Is It Worth It?

Running Kafka on Kubernetes adds operational complexity. There's no getting around that. But the benefits - declarative infrastructure, automated rolling upgrades, standardized deployment pipelines, and the ability to run Kafka alongside your microservices in the same cluster - are real.

The key is using Strimzi (or a similar operator), choosing the right storage class, and building clients that handle the inherent dynamism of a Kubernetes environment. Do those things, and Kafka on Kubernetes works. Skip them, and you'll have a very educational but very painful experience.
