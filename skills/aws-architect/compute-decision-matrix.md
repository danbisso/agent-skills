# Compute decision matrix — break-even, cold start, sizing cheat sheet

Supporting detail for `SKILL.md`. Use when the compute choice is close (Lambda vs container) and
you need to reason about cost crossover, cold starts, or sizing.

## Quick decision flow

```
Run time > 15 min? ──yes──► not Lambda. Heavy+sporadic → Fargate. Constant → ECS/EKS. GPU/big-mem → EC2.
        │ no
Needs GPU or >10 GB RAM? ──yes──► EC2 (or ECS GPU task).
        │ no
Needs persistent connections / warm in-memory state? ──yes──► ECS/EKS service (or Fargate).
        │ no
Traffic spiky / bursty / event-driven / low duty cycle? ──yes──► Lambda.
        │ no (sustained, high duty cycle)
Busy > ~40–50% of the time at scale? ──yes──► price Fargate/ECS vs Lambda; container likely cheaper.
        │ no
Default ──► Lambda.
```

## Cost break-even (Lambda vs Fargate/ECS)

Lambda price ≈ invocations × duration(ms) × memory(GB) (plus per-request fee). Containers price ≈
task/instance-hours regardless of how busy they are.

- **Lambda wins** when duty cycle is low: bursty traffic, long idle troughs, unpredictable spikes,
  or low overall volume. You pay nothing while idle.
- **Containers win** when a worker would be busy a large fraction of the time. Rough heuristic: if
  one Lambda-equivalent of work keeps a CPU busy **more than ~40–50%** of wall-clock, a
  right-sized Fargate task or ECS service usually costs less and avoids per-invocation overhead.
- **Always estimate, don't assume.** Compute monthly: `invocations × avg_duration × memory` for
  Lambda vs `tasks × hours × task_size` for Fargate. Include NAT/data and RDS Proxy costs if in a
  VPC.
- **Memory tuning is CPU tuning on Lambda** — CPU scales with allocated memory. A higher memory
  setting can finish faster and cost *less* total. Profile with AWS Lambda Power Tuning before
  assuming "smaller = cheaper".

## Cold starts

| Factor | Effect | Mitigation |
|---|---|---|
| Package/deps size | bigger = slower init | trim deps, layers, tree-shake, smaller runtime |
| Runtime | JVM/.NET slowest; Go/Rust/Node/Python fast | pick a light runtime for latency-critical paths |
| VPC attachment | adds ENI/Hyperplane setup | stay out of VPC unless reaching private resources |
| Init code | runs once per cold start | move heavy init outside handler; lazy-load |
| Concurrency spikes | many simultaneous cold starts | **Provisioned Concurrency** or **SnapStart** (JVM) |

- Cold starts hurt **user-facing sync** paths most. For async/event-driven work they're usually
  irrelevant — don't over-optimize.
- If you need consistently warm Lambda at predictable load, Provisioned Concurrency erodes the
  cost advantage — that's a signal to re-check whether a container service fits better.

## Concurrency & scaling shape

- **Lambda:** scales per-request near-instantly to account concurrency limits (default 1000,
  raisable). Reserve concurrency to protect downstreams (e.g. an RDS that can't take 1000 conns).
  Watch for throttling (429) under burst.
- **Fargate:** scales at task granularity; task start ~tens of seconds. Good for batch bursts,
  poor for instant sub-second scale-out.
- **ECS/EKS service:** target-tracking / step scaling on CPU/mem/queue depth; keep warm headroom
  for steady throughput.
- **EC2 ASG:** slowest to scale (instance launch minutes); use warm pools or over-provision.

## Statefulness & connections

- Lambda is stateless and short-lived: no reliable in-memory cache across invocations, no durable
  connection pool. Use DynamoDB (HTTP, no pooling) or RDS Proxy for relational.
- Containers can hold warm caches and persistent pools — a real advantage for chatty DB workloads
  or expensive client init.

## When the answer is "split it"

It's valid to mix: Lambda for the spiky API/event edge, Fargate/ECS for the heavy or long-running
worker behind a queue. Recipe 2 (async job processing) in `SKILL.md` is exactly this — fast Lambda
front, container worker for the slow part.
