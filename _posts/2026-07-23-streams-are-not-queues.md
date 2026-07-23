---
layout: post
title: "Streams Are Not Queues"
date: 2026-07-23
categories: [engineering, architecture, fintech]
tags: [kafka, sqs, event-streaming, job-queues, system-design]
excerpt: "A retry storm, a stuck consumer group, and the night our team finally understood why Kafka and SQS aren't interchangeable. Told through a fintech payments incident."
---

## 02:47 UTC

Riya's phone buzzes on the nightstand. An on-call alarm.

**PagerDuty: `payment-retry-consumer-group` — lag 340,000 and climbing.**

She opens her laptop half asleep. 340,000 payments sitting unprocessed. Somewhere in that pile is a retry for a real customer's rent payment, stuck for eleven minutes.

She pulls up the consumer group. `payment.retry.requests`. Six partitions. One of them, partition 3, is wedged. A single payment record with a malformed currency code sits at the front, and every loop iteration throws the same exception. Every message behind it in that partition? Blocked. Waiting. Doing nothing.

![Consumer lag chart showing partition 3 diverging to 340k while partitions 0, 1, 2, 4, 5 remain flat near zero]({{ site.baseurl }}/img/stream-vs-queues/consumer-lag-chart.svg)

The five other partitions are keeping up fine. But partition 3 owns every retry for a chunk of customers (by hash), and it is not moving. Nothing skips the bad record. Nothing routes around it. Nothing dead-letters it.

Six months earlier, the engineer who built this had made a reasonable decision: the team already ran Kafka, already trusted it, already knew how to monitor it. Why stand up a second messaging system for something as small as payment retries?

Riya restarts the consumer, skips the offset by hand, writes an incident report, and goes back to bed at 4:15 AM. In the morning stand-up she says the sentence that starts this whole post:

> "We used a stream where we needed a queue."

---

## Two different questions

Riya had used Kafka for three years. She could explain partitioning to a new hire. She could sketch the topology on a whiteboard from memory. And she still built this, because the trap isn't about knowing Kafka. It's about not noticing that two tools that _look_ similar ("systems that move messages between services") are answering completely different questions.

A **stream** answers: _what happened, in what order, and who else needs to know about it?_

A **queue** answers: _here's a task. Do it once. Tell me if it didn't work. The order doesn't matter much._

That second part is easy to miss. A stream _guarantees_ ordering within a partition. A queue deliberately trades ordering away so that any available worker can grab any available record. That trade-off is why queues can isolate a single poison message while streams can't, and it's also why you'd never want a queue processing your ledger postings, where "debit before credit" matters.

Mix the two up and you don't get a slower system. You get what Riya got at 02:47: a task-shaped problem stuffed into a stream-shaped system, with none of the guardrails a real queue would have provided out of the box.

## What a Kafka partition actually looks like

A Kafka partition behaves like a ledger. Append-only, ordered, retained. When a consumer reads offset 4, that record doesn't disappear. It's still sitting there, and it will keep sitting there until retention expires, regardless of how many consumer groups have already moved past it. Within a single partition, ordering is guaranteed: offset 3 is always processed before offset 4. That's not a nice-to-have for a ledger service reconciling debits and credits. It's a hard requirement.

![Kafka partitions as retained, ordered logs read independently by multiple consumer groups]({{ site.baseurl }}/img/stream-vs-queues/kafka-log-diagram.svg)

That's a powerful property when you need it. `ledger-svc`, `fraud-svc`, and a nightly analytics pipeline can all read the exact same payment events independently, at their own pace, without interfering with each other. You can replay six months of KYC events into a new risk model without touching production traffic. Records are consumed in the sense of _observed_, not _destroyed_.

But the same property is what made Riya's night miserable. A classic consumer group can't pull one record out, hold it aside, and hand it to a different worker if the first one crashes. Instead, each partition is owned by exactly one consumer in the group. Parallelism is capped by partition count, not worker count.

Kafka's own project maintainers described this gap when they wrote KIP-932, the proposal that became "Queues for Kafka." Teams were creating topics with far more partitions than the data actually justified, just to get enough parallel consumers. The consumer group protocol itself had no concept of acknowledging or retrying a single record: everything operated at the partition level. That's exactly the wall Riya hit. Six partitions sized for "should be enough," and one bad record that nobody could skip without SSH-ing into a box.

## What a queue does instead

A queue makes a much narrower promise, and it makes a deliberate sacrifice to get there: it gives up strict ordering. A worker pulls a record. While it's being processed, nobody else can see it (it's checked out, not gone). The worker finishes and deletes it. If the worker crashes before the delete happens, the checkout expires after a timeout and the record becomes visible again for the next available worker — which might not be the same worker, and might process it out of the original sequence. The system counts how many times this has happened, and after a threshold, parks the record in a dead-letter queue.

That lack of ordering is the reason queues _can_ isolate a poison message. Because records aren't chained to each other in sequence, one record failing doesn't hold up the rest. In a stream, ordering and head-of-line blocking are two sides of the same coin. In a queue, any worker can grab any visible record, so a stuck one just sits in timeout while everything else moves on.

![Lifecycle of a queue message: receive, hide, delete on success, automatic redelivery on failure, dead-letter queue after repeated failures]({{ site.baseurl }}/img/stream-vs-queues/queue-lifecycle-diagram.svg)

SQS works this way. RabbitMQ works this way. Even a Postgres table can do it, using `SELECT ... FOR UPDATE SKIP LOCKED`, a locking clause built precisely so that several workers can each claim a different row from a shared job table without blocking each other or grabbing the same row twice.

A classic Kafka consumer group has none of this. No per-record acknowledgment. No automatic redelivery of just the one record that failed while everything else moves on. No delivery-count field to check. You either commit the offset and risk swallowing failures, or you don't commit and risk reprocessing the whole batch behind the stuck one, which is the trap that jammed partition 3.

## Same payment, different jobs

Here's what actually clicked for Riya's team once they sat down and re-drew the architecture: the _same_ payment event legitimately needs to feed both patterns at the same time.

![A fintech architecture where payment events feed a Kafka stream for ledger, fraud scoring, and analytics, while a separate queue handles payment retries and webhook delivery]({{ site.baseurl }}/img/stream-vs-queues/fintech-architecture-diagram.svg)

**`payment.events` on Kafka** is the stream. The ledger service, fraud scoring, and the data warehouse all read the same sequence of events independently, sometimes replaying from the beginning. Multiple readers, ordering matters (debits must land before credits), history matters.

**Payment retries and webhook delivery** are queue work. Each retry is a single task. It should be attempted, acknowledged if it succeeds, retried if it doesn't, and dead-lettered if it keeps failing. One owner per task, no replay needed, and no reason retry #4821 has to finish before retry #4822.

Riya's team had collapsed both needs into one Kafka topic because the broker was already there. The stream half of that decision was sound. The queue half is what paged her.

## What retries actually cost to build in each system

This is where the incident post-mortem got interesting, because "we'll just add retry handling" means very different things depending on what's underneath.

**With SQS**, a dead-letter queue is two fields on the queue's own config:

```json
{
  "RedrivePolicy": {
    "deadLetterTargetArn": "arn:aws:sqs:ap-south-1:111122223333:payment-retry-dlq",
    "maxReceiveCount": 5
  }
}
```

Every record tracks its own receive count automatically. Fail five times, SQS moves that single record into the DLQ. No retry logic, no routing code, no scheduler. Every other record in the queue keeps flowing the whole time.

**With a classic Kafka consumer group**, no such mechanism exists. A partition offset is committed or it isn't; there's no per-record attempt counter in the protocol. So teams end up building the same workaround: a chain of retry topics. Fail on `payment.events`? Produce the record onto `payment.events.retry-5s`. Fail again? `retry-1m`. Then `retry-10m`. Then a topic designated as the DLQ, with your own application code deciding when to give up.

![Kafka's manual retry-topic-chain requiring custom producers, consumers, and counters vs SQS's native RedrivePolicy config]({{ site.baseurl }}/img/stream-vs-queues/retry-complexity-diagram.svg)

Each arrow in that Kafka chain is a producer, a consumer, a schema, and a schedule that your team writes, tests, and pages for. And none of it fixes partition 3's actual problem. The retry-topic pattern moves the _retry_ off the main topic, but a classic consumer group still can't skip one poison record mid-partition without somebody doing it by hand.

|                          | **SQS**                                               | **Kafka (classic consumer group)**                        |
| ------------------------ | ----------------------------------------------------- | --------------------------------------------------------- |
| Ordering                 | Best-effort (standard) or FIFO with throughput limits | Guaranteed within a partition                             |
| Retry mechanism          | Automatic via visibility timeout                      | Manual: produce to retry topics, sleep/reschedule         |
| Attempt tracking         | Built-in `ApproximateReceiveCount`                    | Self-managed header your code writes and checks           |
| Poison-message isolation | Per-record: one bad record never blocks another       | Per-partition: one bad record stalls everything behind it |
| DLQ routing              | Declarative config (`RedrivePolicy`)                  | A topic you create and route to in application code       |
| Custom code required     | None for the retry/DLQ path                           | Producers, consumers, schedulers, counters per hop        |

Now here's the side-by-side of what Riya's night _would_ have looked like if the retry job had been on SQS instead:

![Side-by-side timeline: SQS handles a poison message in 6 minutes automatically with zero pages, while Kafka blocks partition 3 for 99 minutes until a human skips the offset]({{ site.baseurl }}/img/stream-vs-queues/poison-message-timeline.svg)

On SQS, the same malformed record would have bounced through the visibility timeout five times over about six minutes, landed in a DLQ, and been reviewed on Monday. No page, no human, no other record affected. On Kafka, it took a 3 AM wake-up, an SSH session, a manual offset skip, and 99 minutes of blocked payments.

## The honest counterargument: "we already run Kafka"

It would be wrong to frame this as a simple "Kafka bad, SQS good" takeaway. Riya's team wasn't foolish to reach for Kafka. Sticking with infrastructure you already operate, monitor, and trust is a real argument. A second messaging system means more secrets to rotate, more dashboards to build, more failure modes to train new hires on.

And the gap is closing. Kafka 4.2 (released February 2026) shipped **share groups** as a GA feature via KIP-932. Share groups let multiple consumers pull from the same partition cooperatively, and each consumer explicitly acknowledges individual records as `accept`, `release` (put it back for retry), or `reject` (mark it as unprocessable). Delivery attempts are tracked per-record, not per-partition. That's a genuine structural change, and it solves the core isolation problem that burned Riya.

Two caveats, though. First, DLQ support for share groups is still in development (KIP-1191, targeting Kafka 4.4 later this year), so the "reject and automatically route to a dead-letter topic" step still requires some application-level wiring as of today. Second, most existing Kafka deployments are still on classic consumer groups, where the retry-topic chain described above remains the reality. If you're evaluating share groups now, budget time to test their failure modes in your own environment rather than assuming immediate parity with a system that's been running these semantics for close to two decades.

## The decision, simplified

![Decision flowchart: if you need replay, multiple independent readers, or an audit trail, use a stream; if you need a job to run once with retry on failure, use a queue]({{ site.baseurl }}/img/stream-vs-queues/decision-flowchart.svg)

Will more than one system need its own independent read through the same history? Does ordering within a partition matter (ledger reconciliation, fraud sequencing, audit reconstruction)? Then it's a stream: retained, replayable, ordered, multi-reader. Kafka is the right tool.

Is the job really just "do this thing, try again if it fails, give up after N attempts"? Could the tasks run in any order without breaking correctness? Then it's a queue: single-owner, delete-on-success, dead-letter on failure. SQS, RabbitMQ, or a Postgres SKIP LOCKED table will all do it, and they've been doing it for years.

If the answer is "both" (and in a fintech payments system, it usually is), that's not a contradiction. It's a sign you need the same event feeding a stream for observability and a queue for task execution. Two systems, each doing the job it was shaped for.

## Six weeks later

Riya's team kept `payment.events` on Kafka. The ledger, fraud scoring, and analytics services still read it independently, and the fraud team replayed six months of history into a new model without touching production. That part was never the problem.

Payment retries moved to SQS. The DLQ collects whatever fails, reviewed every Monday, nobody paged at 3 AM. The migration was about forty lines of infrastructure-as-code and a week of shadowing both paths in parallel.

The week after cutover, a new malformed currency code came through. The SQS queue bounced it five times, parked it in the DLQ, and nobody's phone buzzed. Riya saw it Monday morning in the review, fixed the upstream validation, and closed the ticket before lunch.

**The system that runs your background jobs right now: did someone choose it for the job, or is it just whatever was already running?**

---

_Opined and edited by me, drafted and visualised with AI. © Shahbaz Ahmed, 2026. Shared under CC BY 4.0 — use it, adapt it, just credit it._
