+++
title = "ELK vs Loki, or How Much Do You Want to Spend on Logs?"
date = 2026-01-08
[taxonomies]
tags = ["observability", "devops", "infrastructure"]
+++

At some point, every team outgrows `kubectl logs` and `grep`. You need centralized logging. You need to search across services, correlate requests, and not lose logs when pods restart.

The two dominant choices are ELK (Elasticsearch + Logstash + Kibana) and the Grafana stack (Loki + Promtail + Grafana). I've run both in production. Here's the honest comparison.

## ELK: The Established Giant

ELK has been the default centralized logging stack for a decade. It's powerful, mature, and well-understood. It's also complex, resource-hungry, and expensive.

The components:

**Elasticsearch** - a distributed search engine that indexes your logs. Every field in every log line is indexed, which makes queries fast and flexible. The trade-off: indexing everything requires significant CPU, memory, and storage.

**Logstash** - the log processing pipeline. It ingests logs from various sources, transforms them (parse, filter, enrich), and writes them to Elasticsearch. It's a Swiss army knife, but it's Java-based and uses more memory than you'd expect.

**Kibana** - the visualization layer. Dashboards, search, log exploration, and the Discover view that everyone actually uses. Kibana is good. I have no complaints about Kibana.

In practice, most teams also use **Filebeat** instead of (or alongside) Logstash. Filebeat is a lightweight log shipper that reads log files and sends them to Elasticsearch or Logstash. It uses a fraction of the resources.

A typical flow: Application writes JSON logs to stdout -> Kubernetes captures them in files -> Filebeat reads the files -> Filebeat sends to Elasticsearch (or Logstash for processing) -> Kibana for querying.

### The Good

**Query power.** Elasticsearch's query DSL is extraordinarily flexible. Full-text search, regex, range queries, aggregations, nested object queries. If your log data has it, you can find it.

**Speed at scale.** With proper index configuration and enough nodes, Elasticsearch handles massive log volumes with sub-second query times. The index-everything approach means queries don't need to scan raw data.

**Ecosystem maturity.** Beats (Filebeat, Metricbeat, etc.), ingest pipelines, index lifecycle management, cross-cluster search - there's a solution for almost every log management need.

**APM integration.** Elastic APM integrates natively, giving you traces, metrics, and logs in one platform.

### The Bad

**Resource consumption.** A production Elasticsearch cluster for moderate log volume (50GB/day) needs at least 3 data nodes, each with 32GB RAM and fast SSDs. That's ~100GB of RAM just for Elasticsearch. Add Logstash and Kibana, and you're looking at significant infrastructure cost.

**Operational complexity.** Shard management, index lifecycle policies, cluster health, split-brain prevention, JVM heap tuning. Elasticsearch requires ongoing care. It's not install-and-forget.

**Storage cost.** Elasticsearch stores the raw data plus the inverted index. The storage overhead is roughly 1.5x-2x the raw log volume. At 50GB/day of logs, you're storing 75-100GB/day in Elasticsearch. Keeping 30 days of logs means 2-3TB of fast storage.

**License changes.** Elastic moved away from Apache 2.0 to SSPL, then Elastic License v2. OpenSearch (the AWS fork) exists as an alternative, but the ecosystem split adds confusion.

## Loki: The Cost-Conscious Alternative

Grafana Loki takes a fundamentally different approach: it does NOT index log content. It only indexes a set of labels (like `{app="order-service", namespace="production"}`). When you query, Loki filters by labels to find the relevant streams, then scans the raw (compressed) log data within those streams.

The components:

**Loki** - the log storage and query engine. Stores compressed log chunks in object storage (S3, GCS, MinIO) with a small index.

**Promtail** (or Grafana Alloy) - the log collection agent. Runs as a DaemonSet in Kubernetes, reads container logs, adds labels, and pushes to Loki.

**Grafana** - the same Grafana you use for metrics. Log exploration is built in with the Explore view and LogQL for querying.

The flow: Application writes JSON logs to stdout -> Promtail reads them, adds Kubernetes labels -> Promtail pushes to Loki -> Grafana for querying.

### The Good

**Cost.** This is Loki's killer feature. Because it doesn't index log content, it uses dramatically less CPU and memory than Elasticsearch. Storage is mostly the compressed log data in cheap object storage (S3). For the same log volume, Loki costs 5-10x less to operate than ELK.

**Simplicity.** Loki in single-binary mode runs on a single node with minimal configuration. For small to medium deployments, there's almost nothing to tune. Even in distributed mode, it's simpler than Elasticsearch cluster management.

**Grafana integration.** If you're already using Grafana for Prometheus metrics, adding Loki is natural. You can correlate metrics and logs in the same dashboard. Click a spike in a metric graph, jump directly to the logs from that time window.

**Label-based filtering.** Labels work like Prometheus labels. `{app="order-service", level="ERROR"}` instantly narrows to the relevant log streams without scanning anything.

**Object storage backend.** Logs go to S3/GCS/MinIO. Cheap, durable, scalable. No managing local SSDs or dealing with Elasticsearch shard rebalancing.

### The Bad

**Limited query capabilities.** LogQL (Loki's query language) is less powerful than Elasticsearch's query DSL. Complex aggregations, nested object queries, and multi-field correlations are either awkward or impossible.

Want to find logs where `response_time > 1000 AND user_id = "abc"`? In Elasticsearch, this is a straightforward boolean query. In Loki, you need to parse the log line at query time:

```
{app="order-service"} | json | response_time > 1000 and user_id = "abc"
```

This works, but it's parsing every log line at query time. For high-volume streams, it's slow.

**Query performance.** Loki's queries scan compressed log data. For narrow time ranges and good label selectivity, it's fast. For broad queries ("find this error in the last 30 days across all services"), it's painful. Elasticsearch would return that in seconds because the content is indexed. Loki might take minutes.

**High cardinality labels are dangerous.** If you add a label with high cardinality (like `userId` or `requestId`), Loki's index explodes. Labels should be low cardinality: service name, namespace, log level. Everything else goes in the log line and is parsed at query time.

**No built-in alerting on log content.** Loki Ruler can alert on log-based metrics, but it's more limited than what you can do with Elasticsearch watchers or Kibana alerts.

## The Cost Comparison

Let me put real numbers on this. For a team running 20 microservices generating ~100GB of raw logs per day, retaining 30 days:

**ELK:**
- 3 Elasticsearch data nodes: 3x m5.2xlarge (8 vCPU, 32GB RAM) = ~$2,100/month
- Storage: 6TB EBS gp3 = ~$480/month
- 1 Kibana + 1 Logstash node: ~$400/month
- **Total: ~$3,000/month**

**Loki:**
- 1-2 Loki instances: 2x m5.xlarge (4 vCPU, 16GB RAM) = ~$350/month
- S3 storage: 1.5TB (compressed) = ~$35/month
- Promtail: DaemonSet, minimal overhead on existing nodes
- Grafana: already running for metrics
- **Total: ~$400/month**

These are rough numbers and your mileage varies. But the order of magnitude difference is real. Loki's approach of "don't index everything" directly translates to lower compute and cheaper storage.

## When Each Makes Sense

**Choose ELK when:**
- You need powerful full-text search across log content
- You're doing compliance or security auditing that requires complex queries
- You have dedicated infrastructure teams to manage Elasticsearch
- You need APM + logging in one platform (Elastic APM)
- Query speed on broad time ranges is critical
- You're already running Elasticsearch for other purposes (application search, etc.)

**Choose Loki when:**
- Cost is a significant concern (it usually is)
- You're already using Grafana for metrics
- Your log queries are mostly "show me logs for service X in the last hour"
- You want operational simplicity
- Your team doesn't have Elasticsearch expertise
- You're running on Kubernetes and want native integration

## My Recommendation

For most teams, especially those already invested in the Prometheus + Grafana ecosystem, Loki is the right default. It covers 80% of logging use cases at 10% of the cost. The remaining 20% - complex analytics, broad searches, compliance queries - can often be handled by exporting specific log data to a data warehouse or using Loki's query capabilities creatively.

If you're starting fresh and don't have strong requirements for complex log analytics, go with Loki. You can always add Elasticsearch later for specific use cases.

If you're already running ELK and it's working, don't rip it out. But if you're feeling the pain of operating Elasticsearch clusters and the cost is climbing, evaluate Loki as a replacement. The migration isn't trivial (you'll lose some query capabilities), but the operational and cost savings are substantial.

## The Hybrid Approach

Some teams run both. Loki for general-purpose logging (high volume, moderate retention). Elasticsearch for security logs and audit trails (lower volume, long retention, complex queries). The ingestion pipeline splits logs by category and routes them to the appropriate backend.

This adds complexity but lets you optimize cost and capability independently. Security logs that need to be retained for a year with full-text search go to Elasticsearch. Application logs that are only useful for a few weeks go to Loki.

There's no wrong answer here, only trade-offs. Understand your query patterns, your volume, your budget, and your team's operational capacity. Then pick the tool that fits. Or pick both. Whatever lets you actually find the problem at 3 AM instead of staring at a loading spinner.
