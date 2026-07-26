# AWS Messaging: SQS, SNS, and EventBridge

Kafka gets the deep-dive treatment in interviews because it has the most mechanics to probe, but the majority of production backend systems on AWS run on SQS, SNS, and EventBridge — and interviewers at AWS-native shops (and AWS itself) will go just as deep on these. The three services solve different problems and are almost always used together, not as alternatives to each other: SQS is the queue, SNS is the fan-out, EventBridge is the router. Knowing which one to reach for, and the specific failure modes each has, is the signal.

## SQS: the managed queue

SQS is a fully managed, at-least-once, point-to-point queue. There is no cluster to run, no partitions to size, no broker to patch — that operational simplicity is the entire pitch, and it is a legitimate one: most background-job workloads do not need Kafka's replay and ordering guarantees, they need "do this work reliably, eventually, without me managing anything."

**Standard vs FIFO** is the first fork in every design:

| | Standard | FIFO |
| --- | --- | --- |
| Ordering | Best-effort, not guaranteed | Guaranteed within a `MessageGroupId` |
| Duplicates | Possible, no dedup | Deduplicated for 5 minutes via `MessageDeduplicationId` (explicit or content-based hash) |
| Throughput | Nearly unlimited | 3,000 msg/s per queue with batching (300 without); scales by adding more message groups |
| Cost | Lower | Slightly higher per request |
| Default choice | Yes — pick this unless you have a specific ordering need | Only when order-per-entity actually matters (e.g., `OrderCreated` must precede `OrderShipped` for the same order) |

The dedup window is the detail that trips people up in interviews: FIFO dedup is 5 minutes, not forever. A message resent 10 minutes after a failed attempt is treated as new. This means FIFO dedup is a safety net for fast retries (network blips, quick redeploys), not a substitute for an idempotent consumer — you still need your own dedup (an `inbox` table keyed on a business event ID) for the general case. This is exactly the same idempotent-consumer requirement covered in `01-messaging-fundamentals.md`; SQS does not change that requirement, it just adds a narrow best-effort layer on top of it.

**Visibility timeout** is SQS's version of Kafka's "commit the offset only after processing." When a consumer receives a message, SQS hides it from other consumers for the visibility timeout (default 30s, configurable up to 12h) rather than deleting it immediately. The consumer must call `DeleteMessage` after successfully processing; if it doesn't (crash, timeout, unhandled exception), the message becomes visible again and another consumer picks it up. Two sharp edges:

- **Timeout too short for the work** → the message reappears and gets processed twice *while the first attempt is still running* — a duplicate caused by your own configuration, not a network failure. Set the timeout comfortably above your p99 processing time, or use `ChangeMessageVisibility` to extend it from within a long-running handler ("heartbeating").
- **Timeout too long** → a genuinely failed message sits invisible for the full window before anyone can retry it, inflating latency on failures.

**Dead-letter queues and redrive**: configure a `RedrivePolicy` with `maxReceiveCount` — after a message is received (and its visibility timeout expires unacknowledged) that many times, SQS automatically moves it to a DLQ instead of redelivering it forever. This is the managed equivalent of the poison-message handling covered in the fundamentals file. The DLQ is a normal queue; nothing consumes it automatically, so you need alerting on `ApproximateNumberOfMessagesVisible` on the DLQ, and a redrive plan — SQS supports `StartMessageMoveTask` to redrive DLQ messages back to the source queue once you've fixed the bug, without writing that plumbing yourself.

**Long polling**: `ReceiveMessage` with `WaitTimeSeconds` up to 20 holds the connection open until a message arrives or the timeout elapses, instead of returning empty immediately (short polling). Long polling is free and should always be on — short polling only burns API calls and money for no benefit, and interviewers treat "why would you ever short-poll" as a quick sanity check.

**Large payloads**: SQS message bodies cap at 256 KB. The standard pattern for larger payloads is the *claim-check pattern*: put the payload in S3, send the S3 key in the SQS message. AWS ships this as the "Amazon SQS Extended Client Library" if you want it off the shelf rather than hand-rolled.

**Lambda as a consumer**: Lambda's SQS event source mapping polls the queue for you and invokes your function with a batch. Two settings matter for correctness: `BatchSize` (how many messages per invocation) and, critically, **partial batch failure handling** — by default, if any message in a batch throws, Lambda's older behavior retried the *whole batch*, redoing already-succeeded work. `ReportBatchItemFailures` fixes this: your function returns which specific message IDs failed, and only those are retried. Forgetting to enable this on a batched Lambda consumer is a classic AWS-specific interview gotcha — "why are messages that already succeeded getting reprocessed?"

## SNS: fan-out

SNS is pub/sub: one message published to a topic can fan out to many subscribers (SQS queues, Lambda, HTTP endpoints, email, mobile push). The reason SQS and SNS are almost always paired rather than used alone: **SQS has no native fan-out** — one message, one consumer, deleted on ack. If two independent services (billing and analytics, say) both need every `OrderConfirmed` event, publishing directly to one SQS queue means only one of them ever sees each message. The fix is the standard **SNS+SQS fan-out** pattern: publish once to an SNS topic, subscribe one SQS queue per consumer. Each subscriber gets its own durable, independently-consumed copy — this is SQS/SNS's answer to what Kafka gets for free from consumer groups reading the same partitioned log.

**Filter policies** let each subscriber declare which messages it wants (by message attribute, e.g. `event_type = "OrderConfirmed"`) so SNS only delivers matching messages to that subscription — avoiding a fan-out queue that every consumer has to filter itself. This is SNS's (much simpler) analogue of Kafka topics-per-event-type, useful when you'd rather have one topic and attribute-based routing than many narrow topics.

**FIFO SNS** exists and pairs with FIFO SQS subscribers to preserve ordering through the fan-out, at the same throughput and dedup-window caveats as FIFO SQS.

## EventBridge: the router

EventBridge is the newest of the three and solves a different problem: routing structured events from many sources to many targets based on content, without every producer and consumer needing to know about each other. An **event bus** receives events (from AWS services, your own applications via `PutEvents`, or 20+ SaaS partner integrations); **rules** match on event pattern (a JSON filter over the event's fields, not just a topic name) and route matches to **targets** — Lambda, Step Functions, SQS, SNS, Kinesis, another event bus, and more.

The practical difference from SNS: SNS filters on message *attributes* you set alongside the payload; EventBridge's rule patterns match on the *event content itself* (nested JSON, numeric ranges, prefix/suffix matching, `anything-but`), which makes it a better fit for routing domain events by business meaning ("route this to the fraud pipeline if `amount > 10000` and `country != home_country`") rather than a flag you have to remember to set at publish time.

**Schema registry**: EventBridge can infer and store a JSON schema per event type from observed traffic, generate bindings, and — the part interviewers care about — this is where the *event schema versioning* problem from `03-event-driven-patterns.md` becomes concrete on AWS: a registered schema plus code bindings makes a breaking change to an event shape visible in review rather than discovered by a downstream consumer crashing.

**EventBridge Pipes** connect a source (SQS, DynamoDB Streams, Kinesis, MSK) directly to a target with optional filtering and a lightweight enrichment step (a Lambda call) in between — no glue Lambda required just to move and reshape events from a stream into a downstream service. This is the AWS-managed answer to "I need a small transform between my CDC stream and my event bus" without standing up Kafka Connect or a Debezium sink connector.

## Putting it together: a fan-out order pipeline

A concrete shape that comes up constantly in system-design rounds at AWS shops:

```
API → (local DB txn + outbox row) → relay → SNS "OrderEvents" topic
                                              ├─ SQS "billing-queue"   → billing service
                                              ├─ SQS "analytics-queue" → analytics pipeline (batched writes to warehouse)
                                              └─ EventBridge bus       → rule: amount > $10k → fraud-review Step Function
```

The outbox writes to SNS once; each downstream team gets an independently-scaled, independently-failing SQS queue with its own DLQ; and anything needing content-based routing (not just "give me all order events") goes through EventBridge instead of a new SNS filter policy per rule. This is the same reliability backbone as the Kafka-based designs in `03-event-driven-patterns.md` — local transaction, outbox, at-least-once relay, idempotent consumers everywhere — with AWS-managed services standing in for what Kafka's partitioned log and consumer groups provide natively. The worked outbox-to-SQS code (the relay, `FOR UPDATE SKIP LOCKED` for concurrent relay instances, and the FIFO dedup mapping) is in `../06-system-design/02-caching-and-microservices.md`.

## When to reach for which (and when to reach for Kafka instead)

- **Background jobs, task queues, decoupling one producer from one consumer type** → SQS Standard. Default choice; add FIFO only when a specific entity's events must apply in order.
- **One event, multiple independent consumer teams** → SNS+SQS fan-out, or EventBridge if routing needs to be content-based rather than "everyone gets everything."
- **Content-based routing across many event types and many targets, especially involving SaaS integrations or multiple AWS services** → EventBridge.
- **Replay, high-volume streaming, stream processing, strict per-key ordering at scale, multiple consumer *groups* re-reading the same history** → none of the above do this well; that's Kafka's (or Kinesis's) job. SQS/SNS/EventBridge are consumed-once-and-gone; there is no "replay last week's events" without building your own archive (e.g., an S3 sink).
- **Mixed workloads are the normal end state**: SQS for jobs, SNS/EventBridge for fan-out and routing, Kafka or Kinesis for the durable event backbone — in the same architecture, not as competing choices. See the broker comparison table in `01-messaging-fundamentals.md` for the full decision matrix including RabbitMQ and NATS.

## Common pitfalls

- Forgetting `ReportBatchItemFailures` on a Lambda SQS consumer — one bad message in a batch causes the *whole batch* to retry, reprocessing already-successful work.
- Visibility timeout shorter than actual processing time — the queue itself creates duplicates, independent of any network failure.
- Using SNS FIFO/SQS FIFO everywhere "to be safe" — throughput ceilings (3,000 msg/s per group) turn into a real bottleneck on high-volume topics that didn't need strict ordering in the first place.
- Treating the FIFO 5-minute dedup window as exactly-once — it isn't; consumers still need their own idempotency key check for correctness beyond fast retries.
- No alerting on DLQ depth — messages land in the DLQ and sit silently until a customer complains, because nothing was watching `ApproximateNumberOfMessagesVisible` on it.
- Reaching for EventBridge's content-based routing when a simple SNS filter policy (or even a second SQS queue) would do — EventBridge adds a moving part that isn't free if all you need is "these three services get a copy."
