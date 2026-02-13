+++
title = "Kafka Producer Tuning: The Settings That Actually Matter"
date = 2026-02-13
[taxonomies]
tags = ["kafka", "performance"]
+++

Kafka producer tuning is one of those topics where the documentation gives you a wall of configuration parameters and zero guidance on which ones matter for your workload. I've spent more time than I'd like staring at producer metrics, tweaking settings, and running benchmarks. Here's the condensed version of what I learned.

The core tension in producer tuning is throughput vs latency. Every optimization that improves one tends to degrade the other. Knowing where your workload sits on that spectrum is the first step.

## batch.size: The Accumulation Buffer

The producer doesn't send each message individually. It accumulates messages into batches per partition, and sends the batch when it's full (or when `linger.ms` expires, whichever comes first).

`batch.size` (default: 16384 bytes, or 16KB) controls the maximum size of a batch in bytes. Larger batches mean more messages per network request, which means better throughput and better compression ratios. Smaller batches mean lower latency because messages don't wait as long to be sent.

For throughput-oriented workloads, I typically set this to 64KB or 128KB:

```properties
batch.size=131072
```

For latency-sensitive workloads where you need messages delivered ASAP, the default 16KB is usually fine, paired with a low `linger.ms`.

One thing that catches people: `batch.size` is per partition. If you're producing to a topic with 24 partitions, the producer is maintaining 24 separate batch buffers. The total memory used for batching is roughly `batch.size * number_of_partitions`, bounded by `buffer.memory`.

## linger.ms: The Patience Setting

`linger.ms` (default: 0) tells the producer how long to wait for a batch to fill before sending it. With the default of 0, the producer sends a batch as soon as it has at least one message. This minimizes latency but usually means you're sending tiny batches.

Setting `linger.ms` to a small value like 5-20ms dramatically improves batching efficiency:

```properties
linger.ms=10
```

With `linger.ms=10`, the producer waits up to 10 milliseconds for more messages to arrive before sending the batch. If the batch fills up before 10ms, it sends immediately. If not, it sends whatever it has after 10ms.

For most workloads, `linger.ms=5` is a sensible default. You're adding at most 5ms of latency in exchange for significantly better batching. The throughput improvement can be 2-5x depending on your message rate.

For bulk data pipelines where latency doesn't matter, I've gone as high as `linger.ms=100`. The batches are consistently full, compression is maximally effective, and throughput is excellent.

## Compression: Free Throughput (Almost)

Compression reduces the size of batches before they're sent over the network and stored on the broker. This means less network bandwidth, less disk I/O on the broker, and better throughput.

```properties
compression.type=lz4
```

The options:

- **none** (default): No compression. Maximum CPU efficiency, maximum network and disk usage.
- **gzip**: Best compression ratio. Highest CPU cost. Good for bandwidth-constrained networks.
- **snappy**: Moderate compression, low CPU. Good general-purpose choice.
- **lz4**: Slightly better compression than Snappy with comparable or better speed. My default recommendation.
- **zstd**: Best compression ratio after gzip, with much better speed. Available since Kafka 2.1. Increasingly the best choice for most workloads.

In our benchmarks with typical JSON-serialized events:

| Compression | Ratio | Produce Throughput | CPU Overhead |
|-------------|-------|-------------------|-------------|
| none        | 1.0x  | baseline          | none        |
| snappy      | 2.1x  | +40%              | low         |
| lz4         | 2.3x  | +45%              | low         |
| zstd        | 2.8x  | +50%              | moderate    |
| gzip        | 3.1x  | +30%              | high        |

The "produce throughput" improvement comes from smaller batches fitting through the network faster. The actual messages-per-second rate goes up because the network is no longer the bottleneck.

zstd gives you the best balance of compression ratio and speed. lz4 is the safe choice if you want minimal CPU impact. gzip is almost never the right answer anymore.

One important note: compression happens at the batch level. Larger batches compress better because there's more redundancy to exploit. This is why `batch.size` and `linger.ms` interact with compression: better batching leads to better compression ratios.

## acks: The Durability Knob

`acks` controls how many broker acknowledgments the producer waits for before considering a write successful.

- **acks=0**: Fire and forget. The producer doesn't wait for any acknowledgment. Maximum throughput, maximum risk of data loss.
- **acks=1**: The leader broker acknowledges the write. If the leader crashes before replicating, the data is lost. Good throughput, some risk.
- **acks=all** (or acks=-1): All in-sync replicas must acknowledge. No data loss (assuming `min.insync.replicas` is properly configured). Lower throughput due to replication latency.

```properties
acks=all
min.insync.replicas=2
```

For any workload where data matters, use `acks=all`. The throughput penalty compared to `acks=1` is typically 10-20%, which is a small price for not losing data.

The interaction with `min.insync.replicas` is critical. If you have `acks=all` and `min.insync.replicas=1`, you're not really getting any additional durability over `acks=1` - if the leader is the only ISR member, `all` just means "the leader." Set `min.insync.replicas=2` with a replication factor of 3 for real durability.

## buffer.memory: The Global Limit

`buffer.memory` (default: 32MB) is the total amount of memory the producer can use for buffering records waiting to be sent. If the buffer is full, `send()` blocks for up to `max.block.ms` (default: 60 seconds) before throwing an exception.

For high-throughput producers, 32MB might not be enough. If your producer is bursty - generating large volumes of messages in short periods - increase the buffer:

```properties
buffer.memory=67108864
```

But be thoughtful. A 64MB buffer per producer instance, across multiple producer instances in a service, adds up. Make sure your container memory limits account for producer buffers on top of heap, RocksDB caches, and other memory consumers.

## Putting It All Together

Here are the configurations I use for two common scenarios:

### High-Throughput Pipeline

```properties
batch.size=131072
linger.ms=50
compression.type=zstd
acks=all
buffer.memory=67108864
enable.idempotence=true
max.in.flight.requests.per.connection=5
```

Optimized for maximum messages-per-second. Accepts up to 50ms of additional latency. Compression is aggressive. Still durable with `acks=all`.

### Low-Latency Event Processing

```properties
batch.size=16384
linger.ms=1
compression.type=lz4
acks=all
buffer.memory=33554432
enable.idempotence=true
max.in.flight.requests.per.connection=5
```

Minimal batching delay. Light compression (lz4 is fast enough to not add meaningful latency). Still durable.

## The Tuning Process

Don't guess. Measure.

1. Start with defaults.
2. Enable producer metrics (Spring Boot Actuator exposes them via Micrometer).
3. Look at `record-send-rate`, `record-size-avg`, `batch-size-avg`, `compression-rate-avg`, `request-latency-avg`.
4. If `batch-size-avg` is much smaller than `batch.size`, your batches aren't filling. Increase `linger.ms`.
5. If `request-latency-avg` is dominated by network round trips, increase `batch.size` and `linger.ms` to reduce request count.
6. If `buffer-available-bytes` is consistently near zero, increase `buffer.memory` or you'll get blocked sends.
7. Try different compression algorithms and measure the impact.

The specific numbers depend entirely on your message size, message rate, network latency, and broker capacity. There's no universal "best" configuration. But the principles - batch more, compress, and measure - apply to every Kafka producer.

Don't ship the defaults to production. And don't copy someone's blog post configuration (including this one) without benchmarking it against your actual workload. The right settings are the ones that match your specific throughput, latency, and durability requirements.
