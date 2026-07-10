# Rate & Connection Limits Cheat Sheet

*System design interview reference — common defaults and hard limits across the stack.*
*Compiled July 2026. Numbers are defaults/documented ceilings; most "defaults" are tunable and real-world limits are usually bound by CPU, memory, or file descriptors first.*

**Column key:** *Rate Limit* = operation/connection/request ceilings (counts per second, concurrent connections). *Bandwidth* = data-size limits (payload/item/object size, MB/sec throughput).

---

## 1. Databases & Caches

| System | Type | Description | Rate Limit | Bandwidth | Notes |
|---|---|---|---|---|---|
| **PostgreSQL** | Relational DB (RDBMS) | ACID relational DB, one process per connection | `max_connections` default **100** (~97 usable, 3 reserved for superusers) | — | Set at server start; use a pooler (PgBouncer) at scale. [1] |
| **MySQL** | Relational DB (RDBMS) | Popular open-source relational DB | `max_connections` default **151** (max 100,000) | — | 150 clients + 1 admin. [2] |
| **Redis** | In-memory KV cache | In-memory key-value store / cache | `maxclients` default **10,000** | — | Capped by OS file-descriptor soft limit. [3] |
| **MongoDB** | Document DB (NoSQL) | Document-oriented NoSQL DB | `maxIncomingConnections` default **65,536** | Max BSON document **16 MB** (hard) | Conns bound by OS `ulimit -n`; use GridFS above 16 MB. [4] |
| **DynamoDB** | Managed KV/Document DB | AWS managed NoSQL key-value/document store | **3,000 RCU / 1,000 WCU** per partition/sec | Item ≤ **400 KB**; partition ≤ **10 GB** | 1 RCU = 1 strong read ≤4 KB; 1 WCU = 1 write ≤1 KB. [5] |
| **Cassandra** | Wide-column DB (NoSQL) | Distributed wide-column store | Recommended **~100,000 rows** per partition | Partition **< 100 MB** (ideal < 10 MB) | Guidelines, not enforced; hard ceiling ~2 B cells/partition. [6] |

---

## 2. Messaging & Streaming

| System | Type | Description | Rate Limit | Bandwidth | Notes |
|---|---|---|---|---|---|
| **Kafka** | Distributed log / stream | Distributed event-streaming log | **~10K–100K+** connections per broker | Default message **1 MB** (`message.max.bytes`) | Conns bound by fds/RAM/handler threads; large messages are an anti-pattern. [7][11a] |
| **RabbitMQ** | Message broker (queue) | AMQP message broker | Connections **unlimited by default** (~10K–100K typical, 500K–1M+ tuned); `channel_max` **2,047** | Max message **128 MiB** | Memory-bound per connection; cap via `connections_max`. [8][11b] |
| **Amazon SQS** | Managed message queue | AWS managed queue (stateless HTTPS) | Standard **~unlimited**/sec; FIFO **300/sec** (3,000 batched) | Payload ≤ **256 KB** | No persistent connections to count; size by request rate. [9] |
| **Amazon Kinesis** | Managed stream | AWS managed data stream (stateless HTTPS) | Per shard **1,000 records/sec** write; **5 GetRecords txns/sec** read | Per shard **1 MB/s** write, **2 MB/s** read | No persistent connections; size by shard count. [10] |

---

## 3. Network, Web Servers & Load Balancers

| System | Type | Description | Rate Limit | Bandwidth | Notes |
|---|---|---|---|---|---|
| **TCP** | Transport protocol | Core transport-layer connection protocol | **65,535** ports/side (NOT total conns); ephemeral range **32,768–60,999** (~28K) | — | Conns identified by 4-tuple → real limit = file descriptors / RAM. [11] |
| **Nginx** | Web server / reverse proxy | Event-driven web server & reverse proxy | `worker_connections` default **512** (packaged **1024**) | — | Total conns ≈ `worker_processes × worker_connections`; `keepalive_timeout` 75s. [12] |
| **HTTP/2** | Application protocol | Multiplexed application-layer protocol | **100** concurrent streams per connection | — | RFC-recommended default; clients assume 100 before SETTINGS. [13] |
| **AWS ALB** | Layer-7 load balancer | AWS managed application load balancer | Auto-scales; **no fixed conn/sec cap** | — | Idle timeout default **60 sec** (range 1–4000). [14] |
| **HAProxy** | Load balancer / proxy | TCP/HTTP load balancer | Global `maxconn` ≈ **`ulimit -n` (usually 1024)** | — | Falls back to 100 only in some auto-computed cases; frontend inherits global. [15] |

---

## 4. Cloud & API Limits (AWS)

| System | Type | Description | Rate Limit | Bandwidth | Notes |
|---|---|---|---|---|---|
| **S3** | Object storage | AWS object storage | **3,500** write + **5,500** read per sec **per prefix** | Max object **50 TB** (single PUT ≤ 5 GB) | Unlimited prefixes → parallelize (10 prefixes ≈ 55,000 GET/sec). [16][17] |
| **AWS Lambda** | Serverless compute (FaaS) | AWS event-driven serverless functions | **1,000** concurrent executions per region (soft) | Sync payload **6 MB**; async **256 KB** | Max timeout **15 min** (hard). [18] |
| **API Gateway** | Managed API gateway | AWS managed API front door | **10,000 RPS** + burst **5,000** (REST) | — | Rate raisable, burst fixed; returns HTTP 429 when exceeded. [19] |

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
