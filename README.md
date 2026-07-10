# Rate & Connection Limits Cheat Sheet

*System design interview reference — common defaults and hard limits across the stack.*
*Compiled July 2026. Numbers are defaults/documented ceilings; most "defaults" are tunable and real-world limits are usually bound by CPU, memory, or file descriptors first.*

---

## 1. Databases & Caches

| System | Metric | Limit / Default | Notes |
|---|---|---|---|
| **PostgreSQL** | `max_connections` (default) | **100** | 3 reserved for superusers (~97 usable). Set at server start; each connection is a process, so use a pooler (PgBouncer) at scale. [1] |
| **MySQL** | `max_connections` (default) | **151** | 150 clients + 1 admin. Max configurable: 100,000. [2] |
| **Redis** | `maxclients` (default) | **10,000** | Capped by OS file-descriptor soft limit. [3] |
| **MongoDB** | Max BSON document size | **16 MB** | Hard limit, enforced on every write. Use GridFS for larger. [4] |
| **MongoDB** | `maxIncomingConnections` | **65,536** (default) | Effectively bound by OS `ulimit -n`. [4] |
| **DynamoDB** | Per-partition throughput | **3,000 RCU / 1,000 WCU per sec** | 1 RCU = 1 strong read of ≤4 KB; 1 WCU = 1 write of ≤1 KB. [5] |
| **DynamoDB** | Max item size | **400 KB** | Includes attribute names + values. [5] |
| **DynamoDB** | Max data per partition | **10 GB** | Table auto-splits partitions beyond this. [5] |
| **Cassandra** | Recommended max partition size | **< 100 MB** (ideal < 10 MB) | Rule of thumb, not enforced. Hard ceiling ~2 billion cells/partition. [6] |
| **Cassandra** | Recommended rows per partition | **~100,000** | Best-practice guideline for performance. [6] |

---

## 2. Messaging & Streaming

| System | Metric | Limit / Default | Notes |
|---|---|---|---|
| **Kafka** | Default max message size | **1 MB** (`message.max.bytes` = 1,048,576) | Match `max.request.size` (producer) and `replica.fetch.max.bytes` (broker) if raised. Large messages are an anti-pattern. [7] |
| **Kafka** | Connections handled (per broker) | **~10K–100K+** | No low default cap; `max.connections.per.ip` effectively unbounded (managed services often set ~1,000/IP). Bound by file descriptors, RAM & request-handler threads. [11a] |
| **RabbitMQ** | Default max message size | **128 MiB** | Configurable via `max_message_size`. [8] |
| **RabbitMQ** | Default `channel_max` | **2,047** | Reduced from unlimited in 3.7.5; 16–128 recommended. [8] |
| **RabbitMQ** | Connections handled (per node) | **Unlimited by default; ~10K–100K typical, 500K–1M+ tuned** | Cap via `connections_max`. Memory-bound (~RAM/CPU per connection); shrink TCP buffers to trade throughput for more connections. [11b] |
| **Amazon SQS** | Max message payload | **256 KB** | Use S3 + pointer for larger. [9] |
| **Amazon SQS** | Standard queue throughput | **~Unlimited** API calls/sec | Nearly unlimited per API action. [9] |
| **Amazon SQS** | FIFO queue throughput | **300 msg/sec** (no batching) / **3,000/sec** (batches of 10) | [9] |
| **Amazon SQS** | Connections handled | **N/A — stateless HTTPS API** | No persistent connections to count; size by throughput/request rate, not conns. [9] |
| **Amazon Kinesis** | Per-shard write | **1 MB/sec or 1,000 records/sec** | Includes partition keys. [10] |
| **Amazon Kinesis** | Per-shard read | **2 MB/sec, 5 GetRecords txns/sec** | Applies to provisioned & on-demand. [10] |
| **Amazon Kinesis** | Connections handled | **N/A — stateless HTTPS API** | No persistent connections to count; size by shard count/throughput. [10] |

---

## 3. Network, Web Servers & Load Balancers

| System | Metric | Limit / Default | Notes |
|---|---|---|---|
| **TCP** | Port field width | **16-bit → 0–65,535** | This caps ports *per side*, NOT total connections. [11] |
| **TCP** | Connections identified by | **4-tuple** {src IP, src port, dst IP, dst port} | A server on one port handles far more than 65 K conns; real limit = file descriptors / RAM. [11] |
| **TCP** | Linux ephemeral port range (default) | **32,768–60,999** (~28 K ports) | IANA-recommended: 49,152–65,535. Matters for outbound/client connections. [11] |
| **Nginx** | `worker_connections` (default) | **512** (packaged binaries set **1024**) | Total conns ≈ `worker_processes × worker_connections`. [12] |
| **Nginx** | `keepalive_timeout` (default) | **75 sec** (client), often shown as 65 | [12] |
| **HTTP/2** | `SETTINGS_MAX_CONCURRENT_STREAMS` | **100** (typical default) | RFC-recommended minimum; clients assume 100 before SETTINGS arrives. [13] |
| **AWS ALB** | Idle timeout (default) | **60 sec** | Range 1–4000 sec. Scales automatically; no fixed conn/sec ceiling. [14] |
| **HAProxy** | Global `maxconn` (default) | **≈ `ulimit -n` (usually 1024)** | Falls back to 100 only in certain auto-computed edge cases. Frontend `maxconn=0` inherits global. [15] |

---

## 4. Cloud & API Limits (AWS)

| System | Metric | Limit / Default | Notes |
|---|---|---|---|
| **S3** | Request rate per prefix | **3,500 PUT/COPY/POST/DELETE + 5,500 GET/HEAD per sec** | Unlimited prefixes → scale by parallelizing (10 prefixes ≈ 55,000 GET/sec). [16] |
| **S3** | Max object size | **50 TB** (raised from 5 TB in Dec 2025) | Single PUT ≤ 5 GB; use multipart above that. [17] |
| **AWS Lambda** | Default concurrent executions | **1,000 per region** | Soft limit, raisable via support. [18] |
| **AWS Lambda** | Sync invocation payload | **6 MB** | Hard limit. Async payload limit is 256 KB. [18] |
| **AWS Lambda** | Max execution timeout | **15 min** | Hard limit. [18] |
| **API Gateway** | Default account throttle (REST) | **10,000 RPS, burst 5,000** | Rate raisable; burst is fixed. Returns HTTP 429 when exceeded. [19] |

---

## Quick mental-math anchors for interviews

- **1 KB** item, **1 WCU** = 1 write/sec (DynamoDB); **4 KB**, **1 RCU** = 1 strong read/sec.
- **1 MB/s per Kinesis shard** in, **2 MB/s** out — size shard count from throughput.
- **256 KB** is the SQS payload ceiling; **1 MB** is the Kafka message default; **16 MB** is the MongoDB doc ceiling.
- Connection counts almost always bottleneck on **file descriptors / RAM**, not the protocol.
- Most database "max connections" defaults are **low** (100–151) → assume a **connection pooler** in any design.

---

## Sources

1. PostgreSQL — [Connections and Authentication docs](https://www.postgresql.org/docs/current/runtime-config-connection.html)
2. MySQL — [max_connections tuning (Releem)](https://releem.com/docs/mysql-performance-tuning/max_connections) / MySQL 8.0 Reference Manual
3. Redis — [Client handling reference](https://redis.io/docs/latest/develop/reference/clients/)
4. MongoDB — [Limits and Thresholds](https://www.mongodb.com/docs/manual/reference/limits/)
5. DynamoDB — [Service Quotas](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/ServiceQuotas.html) / [Alex DeBrie: DynamoDB Limits](https://www.alexdebrie.com/posts/dynamodb-limits/)
6. Cassandra — [DataStax: Calculating partition size](https://docs.datastax.com/en/cassandra-oss/2.2/cassandra/planning/planPlanningPartitionSize.html)
7. Kafka — [Confluent: Kafka message size limit](https://www.confluent.io/learn/kafka-message-size-limit/)
8. RabbitMQ — [Configurable Limits](https://www.rabbitmq.com/docs/limits) / [Channels](https://www.rabbitmq.com/docs/channels)
9. Amazon SQS — [Message quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-messages.html) / [FIFO queue quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-fifo.html)
10. Amazon Kinesis — [Quotas and limits](https://docs.aws.amazon.com/streams/latest/dev/service-sizes-and-limits.html)
11. TCP — [Microsoft: Port exhaustion troubleshooting](https://learn.microsoft.com/en-us/troubleshoot/windows-client/networking/tcp-ip-port-exhaustion-troubleshooting) / [The Ephemeral Port Range (ncftp)](https://www.ncftp.com/ncftpd/doc/misc/ephemeral_ports.html)
11a. Kafka connections — [Broker configs (`max.connections`, `max.connections.per.ip`)](https://kafka.apache.org/43/configuration/broker-configs/)
11b. RabbitMQ connections — [Networking](https://www.rabbitmq.com/docs/networking) / [Configurable limits](https://www.rabbitmq.com/docs/limits)
12. Nginx — [Tuning NGINX for Performance (F5)](https://www.f5.com/company/blog/nginx/tuning-nginx)
13. HTTP/2 — [RFC 7540 / nghttp2 discussion](https://github.com/nghttp2/nghttp2/issues/1289)
14. AWS ALB — [Edit load balancer attributes](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-load-balancer-attributes.html)
15. HAProxy — [Protect servers with connection limits](https://www.haproxy.com/blog/protect-servers-with-haproxy-connection-limits-and-queues)
16. S3 — [Optimizing S3 performance](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html)
17. S3 — [Max object size increased to 50 TB](https://aws.amazon.com/about-aws/whats-new/2025/12/amazon-s3-maximum-object-size-50-tb/)
18. AWS Lambda — [Lambda quotas](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)
19. API Gateway — [Request throttling](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html)
