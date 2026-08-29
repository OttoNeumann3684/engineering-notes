# Daily Report Email: A Simple Rate-Limited Sending Queue Architecture

Short answer: schedule one daily report job per recipient, put those jobs in a durable queue, and let a small worker pool push them to the subscriber's public HTTPS endpoint through one shared rate limiter. Scale against the age of the oldest job and a delivery deadline, not the size of the scheduler's initial burst. Don't make the scheduler send email itself.

This is a latency-versus-cost decision. The queue absorbs the morning spike; the limiter protects the receiver; the worker count determines how much parallel rendering and network waiting the system can hide. In an edtech reporting flow, that boundary matters because a school can expect reports within a window even though the downstream endpoint accepts requests at its own pace.

Keep the contract small.

## How should a daily report email queue push to a rate-limited public HTTPS endpoint?

Start by defining one logical report, not by choosing a scheduler. Give every report a stable identity such as `report:{report_date}:{subscriber_id}` and store the reporting window, recipient reference, template version, and delivery deadline with it. The daily producer creates that intent once. A worker claims it, waits for the shared limiter, renders or loads the content, and pushes it to the endpoint. The result is recorded before the lease is released. If the endpoint accepts an idempotency key, use the stable report ID; if it doesn't, the sender and receiver need an explicit duplicate policy.

The queue is the pressure boundary. A task-queue model separates clients that publish work from workers that consume it, as described in the Celery introduction, and multiple workers can process tasks concurrently. That makes a burst visible as backlog rather than forcing the daily trigger to hold open every delivery. It does not make the endpoint faster. Every process must obey one aggregate admission policy; three workers with three independent full-rate limiters can multiply the intended request rate.

A report should move through named states: created, ready, leased, accepted, retryable, permanently rejected, or late. The exact labels can differ, but ambiguous ownership cannot. A public HTTPS subscriber endpoint is also a trust boundary, so authenticate incoming or outgoing messages using the endpoint owner's documented mechanism. I'm not sure which signature algorithm belongs in a vendor-neutral example because that contract is not specified by the sources here; settle it from the receiver's primary documentation and test signed fixtures before launch.

## Run the queue model before choosing capacity

This Python model makes the scheduling behavior visible without pretending an in-memory heap is a production queue. Its concurrency and rate values are example inputs for an eval, not measured recommendations. The transport is a function boundary, so the notebook can exercise ordering and retries while production replaces storage, leasing, and sending.

```python
import asyncio
import heapq
import time
from dataclasses import dataclass, field
from typing import Awaitable, Callable


@dataclass(order=True)
class Job:
    ready_at: float
    job_id: str = field(compare=False)
    attempt: int = field(default=0, compare=False)


class RateLimited(Exception):
    pass


class SharedStartLimiter:
    def __init__(self, starts_per_second: float) -> None:
        self.interval = 1.0 / starts_per_second
        self.next_start = 0.0
        self.lock = asyncio.Lock()

    async def wait(self) -> None:
        async with self.lock:
            now = time.monotonic()
            delay = max(0.0, self.next_start - now)
            if delay:
                await asyncio.sleep(delay)
            self.next_start = max(now, self.next_start) + self.interval


async def drain(
    jobs: list[Job],
    send: Callable[[Job], Awaitable[None]],
    worker_count: int,
    starts_per_second: float,
    max_attempts: int,
) -> None:
    pending = jobs[:]
    heapq.heapify(pending)
    queue_lock = asyncio.Lock()
    limiter = SharedStartLimiter(starts_per_second)

    async def worker() -> None:
        while True:
            async with queue_lock:
                if not pending:
                    return
                job = heapq.heappop(pending)

            delay = max(0.0, job.ready_at - time.monotonic())
            if delay:
                await asyncio.sleep(delay)
            await limiter.wait()

            try:
                await send(job)
            except RateLimited:
                job.attempt += 1
                if job.attempt >= max_attempts:
                    continue
                delay = min(60.0, 2.0 ** job.attempt)
                job.ready_at = time.monotonic() + delay
                async with queue_lock:
                    heapq.heappush(pending, job)

    await asyncio.gather(*(worker() for _ in range(worker_count)))


async def accepted_transport(job: Job) -> None:
    await asyncio.sleep(0.02)


now = time.monotonic()
batch = [Job(now, f"report:2026-08-12:student-{number}") for number in range(20)]
asyncio.run(
    drain(
        batch,
        send=accepted_transport,
        worker_count=3,
        starts_per_second=4.0,
        max_attempts=5,
    )
)
```

The ownership boundary is the useful part: all workers share the limiter, and a retry returns to scheduling instead of sleeping while it owns a worker slot. Production storage must add durable jobs and an atomic claim or lease. A stopped worker's expired lease should make the same job eligible again, which means execution can occur more than once. The stable job ID is therefore part of correctness, not a logging convenience. This is also where notebook-to-prod tests earn their keep — run identical fixture jobs through the model and the production adapter, then compare state transitions rather than accepting similar-looking logs.

## The delivery ledger is the architecture

Most queue failures are accounting failures in disguise. Consider one report whose worker receives an ambiguous network outcome after sending: the receiver may have accepted the payload even though the sender did not record acceptance. Retrying immediately can duplicate the email; suppressing the retry can lose it. The queue cannot infer which world it is in, so the design needs stable identity, bounded retry, and reconciliation with receiver evidence where that evidence exists. Now add a process restart. The lease expires and another worker claims the same logical report. Now add the daily trigger running twice. A uniqueness constraint on the reporting window and subscriber prevents a second logical job before workers even see it. These aren't three unrelated edge cases. They are the same question asked at three boundaries: which durable record proves ownership of this report? Write that answer into the schema, preserve attempt history, and make the final state queryable. A plain ledger also gives support and operations something better than log archaeology: they can distinguish a report that was never scheduled, one waiting behind the limiter, one accepted by the endpoint, and one that exhausted its policy.

Retry only outcomes that may change. A rate-limit response can return to the queue after bounded exponential backoff. Invalid recipient data should enter a terminal review state rather than consume more sending capacity. Backoff spaces attempts farther apart, but it isn't permission to retry forever. Set both an attempt budget and an elapsed-time budget against the report's usefulness window; once that window closes, record a late state explicitly.

No silent churn.

Subscriber or delivery events may arrive after immediate acceptance and may be repeated. Handle them idempotently against the same report identity, acknowledge valid events quickly, and keep their processing capacity separate from the sending pool. Otherwise, a slow event handler can look like slow email delivery even while workers are draining normally.

The catch is that this simple queue architecture is not suitable when every report must arrive at an exact instant; a downstream rate limit makes a burst-time guarantee impossible. Spread job creation earlier or obtain enough downstream capacity for that promise. A plain queue is also a poor fit for reports with long, multi-step dependencies and human approvals, where a workflow system's explicit state transitions may be worth the operational weight. Stick with the queue when jobs are short, independently retryable, and governed mainly by throughput.

## How much worker capacity does the delivery window justify?

Start from the delivery window. For `N` ready reports and an endpoint allowance of `R` starts per second, `N / R` seconds is the admission floor before rendering overhead, network latency, or retries. More workers can hide waiting and keep permitted starts busy, but they cannot beat that floor. Once limiter utilization is consistently full, extra workers mostly add leases, connections, memory, and cost.

| Eval signal | What it reveals | Decision |
| --- | --- | --- |
| Oldest ready-job age rises | The batch risks missing its window | Start earlier or raise safe capacity |
| Limiter is full while jobs wait | The endpoint allowance is binding | Don't add workers for throughput |
| Limiter is idle while jobs wait | Rendering, claims, or concurrency is binding | Profile, then test more workers |
| Attempts per completion rise | Retries are consuming the send budget | Reclassify outcomes or lengthen backoff |
| Event lag rises alone | Feedback handling is behind | Scale that path separately |

Use a fixed synthetic batch and replay it across worker counts. Inject documented retryable outcomes, ambiguous timeouts, duplicate triggers, and worker termination; then measure p50 and p95 oldest-job age, attempts per completed report, completion before the deadline, and total worker-seconds. I would reject a faster configuration if it meets the latency target only by multiplying attempts, because attempt count is real work. For AI-generated reports, persist the evaluated render before delivery so a transport retry doesn't rerun retrieval, consume more prompt budget, or change the content. Your mileage may vary with render time and receiver latency, which is precisely why the concurrency number should come from this harness rather than a copied deployment example.

Before deployment, verify unique batch creation, durable enqueue, atomic claims, lease expiry, bounded retries, idempotent event handling, and an alert tied to oldest-job age. During a rollout, stop new claims before replacing workers and give in-flight sends a bounded drain period. After each daily batch, reconcile created, accepted, late, and permanently rejected counts. The simplest useful operating loop is short: fixture, eval, deploy, observe, adjust.

Measure first.

## References

- https://docs.celeryq.dev/en/stable/getting-started/introduction.html
- https://en.wikipedia.org/wiki/Exponential_backoff
