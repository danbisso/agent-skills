---
name: aws-architect
description: Pick the right AWS backend building blocks for a workload — compute (Lambda vs Fargate vs ECS/EKS vs EC2), orchestration (Step Functions Standard/Express vs queue chaining), eventing/scheduling (EventBridge vs SNS vs SQS vs Kinesis), API layer, and data store — and justify the choice. Use when designing or reviewing a serverless/event-driven AWS backend, choosing compute or orchestration, or resolving "Lambda or container?", "Step Functions or SQS?", "EventBridge or SNS?" questions before implementation.
---

# AWS architect — choosing the right building blocks

Your job here is **selection and justification**, not implementation. Decide the compute,
orchestration, eventing, API, and data services for a workload, defend the choice against the
alternatives, and flag anti-patterns. Once the design is settled, hand off to **`aws-cdk`** to
build it.

Scope boundary: AWS is for **backend and heavier workloads** (durable compute, stateful data,
multi-step workflows, GPU/large-memory). For edge/frontend runtime — request-time logic close to
the user, lightweight KV/SQLite — use **Cloudflare Workers/D1** (see `frontend-dev`), not Lambda.

## How to run a decision

1. **Characterize the workload first.** Pin down: execution duration, traffic shape (spiky vs
   steady vs scheduled), concurrency model, statefulness, payload size, memory/GPU needs,
   latency tolerance (cold-start sensitivity), and expected request volume. Most wrong AWS
   choices come from skipping this step.
2. **Pick compute** using the table below.
3. **Pick the coordination model** — single function, queue chain, or workflow engine.
4. **Pick the event/ingress mechanism** — sync API, async event, stream, or schedule.
5. **Pick data stores** to fit the access pattern, not the other way around.
6. **Layer in** idempotency, DLQs, observability, IAM least-privilege, VPC decision.
7. **Hand off to `aws-cdk`.** State the services + their wiring; let CDK express it.

---

## Decision table 1 — Compute

| Dimension | **Lambda** | **Fargate (ECS/EKS task)** | **ECS/EKS service (long-running container)** | **EC2** |
|---|---|---|---|---|
| Max run time | **15 min hard ceiling** | unbounded | unbounded | unbounded |
| Scaling shape | per-request, instant, to 1000s | task-level, ~seconds | task/pod auto-scaling | ASG, minutes |
| Idle cost | **zero** (pay per ms) | per running task | per running task | per running instance |
| Cold start | 100ms–2s (worse in VPC / large deps) | ~30–60s task start | none (warm pool) | none |
| Statefulness | stateless only | stateless-ish (ephemeral disk) | can hold warm state/caches | full control |
| Connection pooling | hard — use RDS Proxy / DynamoDB | possible | **easy** (persistent pool) | easy |
| GPU / >10GB RAM | no (cap 10GB RAM, no GPU) | limited GPU on ECS | **yes** | **yes** |
| Ops burden | lowest | low | medium (cluster, capacity) | highest (patching, AMIs) |
| Best traffic profile | spiky / bursty / event-driven | batch jobs, sporadic heavy tasks | steady throughput, persistent | predictable steady, special hw |

### Choose-when / avoid-when

- **Lambda — choose when:** event-driven or request/response under 15 min; spiky or unpredictable
  traffic; you want zero idle cost and zero server ops; glue between AWS services; webhook handlers,
  S3/DynamoDB triggers, API backends, cron jobs.
  **Avoid when:** runs >15 min; needs persistent connections or in-process state; sustained
  high-throughput steady load (Lambda gets *more* expensive than containers past a break-even);
  needs GPU or >10GB RAM; long-polling/streaming consumers (you pay for idle wait time).
- **Fargate — choose when:** containerized job that may exceed 15 min but doesn't run constantly
  (batch ETL, video transcode, scheduled heavy task); you want containers without managing a
  cluster; bursty container workloads.
  **Avoid when:** sub-second latency cold-start matters (task start is slow); workload is constant
  enough that reserved EC2 capacity is cheaper.
- **ECS/EKS long-running service — choose when:** steady high request volume; need warm caches /
  connection pools / in-memory state; existing container stack; sidecars/service mesh; want a
  break-even cost win over Lambda at sustained scale.
  **Avoid when:** traffic is spiky with long idle troughs (you pay for idle); team can't carry
  cluster ops; a few Lambdas would do.
- **EC2 — choose when:** GPU/ML, very large memory, specialized kernels/networking, licensing tied
  to instances, or lift-and-shift before refactor.
  **Avoid when:** spiky low-volume work (idle instance burn — classic anti-pattern); anything that
  fits Lambda/Fargate with less ops.

**Cost intuition:** Lambda wins at low/spiky volume and zero-idle; containers win at sustained high
utilization. The crossover is roughly "is the function busy more than ~40–50% of the time?" — past
that, price out Fargate/ECS. Don't guess; estimate invocations × duration × memory vs task-hours.

---

## Decision table 2 — Orchestration & coordination

| Need | **Step Functions Standard** | **Step Functions Express** | **SQS / queue chaining** | **EventBridge (choreography)** |
|---|---|---|---|---|
| Workflow duration | up to 1 year | up to 5 min | unbounded (per-message) | unbounded |
| Pricing model | **per state transition** (pricey at high volume) | per request + duration (cheap, high volume) | per request (cheapest) | per event published |
| Execution history | full, durable, visual | none persisted (logs only) | none | none |
| Exactly-once semantics | yes | at-least-once | at-least-once | at-least-once |
| Human-in-the-loop / wait states | **yes** (waitForTaskToken, days) | no | manual | no |
| Fan-out / Map | **yes** (Map, distributed Map) | yes | manual fan-out via SNS | manual |
| Built-in retry / catch / compensation (saga) | **yes** | yes | DIY | DIY |
| Best for | complex, auditable, long, branching | high-volume short request pipelines | simple decoupling, buffering, smoothing | loosely-coupled event reactions |

### Choose-when / avoid-when

- **Step Functions Standard — choose when:** multi-step business workflow with branching, retries,
  compensation (sagas), human approval / wait-for-callback, long-running (hours–days), and you want
  a durable visual audit trail.
  **Avoid when:** a trivial 2-step flow (just chain Lambdas or use SQS — Standard's per-transition
  cost adds up fast); very high-volume short-lived pipelines (use Express).
- **Step Functions Express — choose when:** high-volume (thousands/sec), short (<5 min), idempotent
  orchestration — e.g. streaming-event processing pipelines, IoT ingestion. Far cheaper per run
  than Standard.
  **Avoid when:** you need durable execution history, exactly-once, or runs over 5 min.
- **SQS / queue chaining — choose when:** decoupling producers from consumers, buffering bursts,
  smoothing load, simple A→queue→B handoffs, retry via redrive + DLQ. Cheapest coordination.
  **Avoid when:** the flow has real branching/state/compensation logic — don't hand-roll a workflow
  engine in queue glue; reach for Step Functions.
- **EventBridge choreography — choose when:** many independent services react to domain events with
  no central controller; you want to add consumers without touching producers.
  **Avoid when:** you need end-to-end visibility of a transaction's progress or ordered
  compensation — choreography hides the overall flow; prefer orchestration (Step Functions).

**Durable-execution note:** "durable function" patterns (a workflow expressed as ordinary code that
checkpoints/replays) are what Step Functions provides natively on AWS via task tokens and durable
state. If you want code-first orchestration without authoring ASL, that's a *how-to-implement*
decision for `aws-cdk`/SDK choice — the architectural pick is still: **Standard** for durable
long/branching workflows, **Express** for high-volume short ones, **SQS** for plain chaining.

**Orchestration vs choreography rule of thumb:** orchestration (Step Functions) when one
transaction's success/failure/compensation must be tracked centrally; choreography (EventBridge/SNS)
when services are autonomous and you optimize for decoupling and independent evolution.

---

## Eventing & scheduling

| Service | Model | Fan-out | Ordering | Replay/retention | Choose when |
|---|---|---|---|---|---|
| **SQS** | point-to-point queue | no (1 consumer group) | FIFO option | up to 14 days | buffer, decouple, smooth bursts, retry + DLQ |
| **SNS** | pub/sub topic | yes (push to many) | FIFO option | none (no replay) | broadcast to known subscribers, SQS fan-out |
| **EventBridge bus** | pub/sub with content routing | yes (rules per target) | no | archive + replay | route domain events by content; SaaS/partner events; schema registry |
| **EventBridge Pipes** | source→filter/enrich→target | n/a | source-dependent | n/a | point-to-point integration without glue Lambda |
| **Kinesis Data Streams** | ordered shard log | many consumers | **per-shard order** | up to 365 days | high-throughput ordered streaming, multiple independent readers, replay |

- **SNS vs EventBridge:** SNS = simple, low-latency, high-throughput fan-out to *known* subscribers.
  EventBridge = content-based routing, many sources (incl. AWS service + SaaS events), schema
  registry, archive/replay — richer but higher per-event cost and slightly higher latency. Default
  to EventBridge for app/domain events; use SNS for high-fan-out broadcast or SQS-fan-out.
- **SQS vs Kinesis:** SQS when you just need a buffer and per-message independence. Kinesis when you
  need **ordering**, **multiple independent consumers** of the same data, **replay**, or sustained
  high-throughput streams. Don't use Kinesis as a plain task queue (over-engineered, shard ops).
- **EventBridge Pipes:** prefer over a hand-written "Lambda that reads from X and writes to Y" when
  the logic is just filter + light transform between a source and a target.

### Scheduling

| Use | Pick |
|---|---|
| One-off or recurring schedule, millions of schedules, per-schedule target | **EventBridge Scheduler** (default for new work) |
| Legacy cron rule already on an event bus | CloudWatch Events / EventBridge rule cron (fine, but Scheduler is the modern choice) |
| Sub-minute / dynamic / app-controlled timing | cron-in-app: a long-running consumer, or Step Functions `Wait`, or DynamoDB TTL trigger |

- **Choose EventBridge Scheduler** for almost all scheduled work: one-time + recurring, time zones,
  flexible windows, scales to millions, decoupled from any bus. Avoid `cron` inside a Lambda loop
  (you pay for sleep and lose at-least-once delivery guarantees).

---

## Supporting decisions (they shape the core choice)

### API / ingress layer

| Option | Choose when | Avoid when |
|---|---|---|
| **API Gateway HTTP API** | REST-ish JSON APIs, JWT auth, lowest cost/latency of the gateways | you need request validation, API keys/usage plans, WAF-per-method, edge caching |
| **API Gateway REST API** | need usage plans, API keys, request/response transforms, fine-grained throttling, WAF | cost/latency-sensitive simple APIs (use HTTP API) |
| **ALB → Lambda/Fargate** | container services, header/path routing, existing ALB, sticky long connections | pure serverless JSON API with no LB need (gateway is simpler) |
| **Lambda Function URL** | single-function webhook/endpoint, no gateway features needed | you need auth/routing/throttling across many routes |
| **AppSync (GraphQL)** | client-driven GraphQL, real-time subscriptions, multiple data sources merged | simple REST CRUD (overkill) |

### Data store fit

| Store | Choose when | Avoid when |
|---|---|---|
| **DynamoDB** | known key-based access patterns, single-digit-ms, serverless scale, high write throughput, TTL/streams | ad-hoc queries, complex joins, strong relational integrity |
| **Aurora Serverless v2** | relational + variable/spiky load, want autoscaling capacity, SQL with joins | ultra-spiky-to-zero with cold tolerance (v2 has a min ACU floor); pure KV |
| **RDS (provisioned)** | steady relational load, predictable capacity, mature SQL features | bursty/zero-idle workloads (idle instance burn) |
| **S3** | objects/blobs, data lake, event source, cheap durable storage, static assets | low-latency record lookups or transactional state |

- **Lambda + relational DB:** always front RDS/Aurora with **RDS Proxy** to avoid connection
  exhaustion from concurrent invocations. DynamoDB sidesteps this entirely (HTTP, no pooling).

### Sync vs async

- **Sync** (API Gateway → Lambda → response) when the caller needs the result now and the work fits
  the timeout (gateway integration timeout 29s; keep well under). 
- **Async** (queue/event + worker, return 202 + status lookup or webhook) when work is slow,
  spiky, or must survive downstream failure. Default to async for anything that can take >a few
  seconds or fan out.

### Cross-cutting requirements (don't ship without deciding these)

- **Idempotency:** any at-least-once consumer (SQS, SNS, EventBridge, Kinesis) can redeliver. Make
  handlers idempotent — idempotency key + dedup store (DynamoDB conditional write), or natural
  upsert. Mandatory for payment/order/email side-effects.
- **Dead-letter queues:** attach a DLQ to every async consumer (SQS redrive policy, Lambda async
  DLQ/on-failure destination, EventBridge target DLQ). Alarm on DLQ depth > 0.
- **Observability:** structured JSON logs; **EMF** to emit metrics from logs without extra calls;
  **X-Ray** tracing across the async chain; alarm on error rate, throttles, DLQ depth, and (for
  Step Functions) failed executions.
- **IAM least-privilege:** one role per function/task, scoped to the exact resources/actions. No
  wildcard `*` resources in production.
- **VPC or not (Lambda):** stay **out** of a VPC unless you must reach private resources (RDS,
  ElastiCache, internal services) — VPC adds cold-start/ENI overhead and NAT cost. If you only call
  AWS APIs and public endpoints, no VPC.

---

## Pattern recipes (well-architected serverless)

1. **Event-driven ingestion.**
   `API Gateway HTTP API` → Lambda (validate, write) → DynamoDB → DynamoDB Streams → EventBridge →
   downstream consumers. Idempotent writes; DLQ on each consumer. Use when ingest must be cheap,
   spiky-tolerant, and fan out to multiple reactions.

2. **Async job processing.**
   `API` returns **202** + job id → publishes to **SQS** → worker (Lambda if <15 min, else Fargate
   task) → writes result + status to DynamoDB → optional SNS/EventBridge "job done" event. DLQ +
   redrive on the queue. Use for slow/heavy work behind a fast API.

3. **Scheduled batch.**
   **EventBridge Scheduler** → Step Functions (**Distributed Map** over the batch) → Fargate tasks
   for the heavy per-item work → aggregate results to S3/DynamoDB. Use for nightly/periodic large
   batch with parallelism and retry/visibility.

4. **Webhook handling (3rd-party callbacks).**
   **Lambda Function URL** or HTTP API → verify signature → enqueue to **SQS** immediately, return
   200 fast → worker processes async. Never do heavy work in the webhook request path (providers
   retry on slow responses → duplicates; idempotency required).

5. **Fan-out / aggregate (map-reduce).**
   Trigger → Step Functions **Map** (or SNS→many SQS) fans work to N parallel workers → results to a
   DynamoDB/S3 collector → aggregation step emits a completion event. Use Step Functions Map when
   you need per-branch retry/visibility; SNS fan-out when workers are fully independent.

6. **Streaming pipeline (high volume, ordered).**
   Producers → **Kinesis Data Streams** → Lambda or Step Functions **Express** per batch → sink
   (S3/DynamoDB/OpenSearch). Use when ordering, replay, and multiple consumers matter.

---

## Anti-patterns — call these out in review

- **Lambda for long-polling / streaming waits** — you pay for idle wait and hit the 15-min wall.
  Use a container consumer or WebSocket API + push.
- **Step Functions Standard for a trivial 2-step flow** — per-transition cost and orchestration
  overhead for nothing. Chain Lambdas or use SQS.
- **EC2 for spiky low-volume work** — idle instance burn. Use Lambda/Fargate.
- **Lambda directly opening RDS connections at scale** — connection exhaustion. Use RDS Proxy or
  DynamoDB.
- **Kinesis used as a plain task queue** — shard management overhead with no ordering/replay need.
  Use SQS.
- **cron inside a long-lived Lambda loop** — paying for sleep, losing delivery guarantees. Use
  EventBridge Scheduler.
- **No DLQ / no idempotency on at-least-once consumers** — silent message loss and duplicate
  side-effects. Always add both.
- **Lambda in a VPC "to be safe"** — needless cold-start + NAT cost when no private resource is
  accessed.
- **Choreography for a transaction that needs end-to-end tracking** — you lose visibility and
  ordered compensation. Use Step Functions.

---

## Output of this skill

Produce: chosen compute + why (vs the runner-up), coordination model, event/ingress mechanism, data
stores, and the cross-cutting decisions (idempotency, DLQ, VPC, IAM scope, observability). Then
**hand off to `aws-cdk`** to implement the wiring.

See `compute-decision-matrix.md` for an expanded compute break-even and cold-start cheat sheet.
