# Node.js Daily Report Email Queue Design: Batch Size, Limits, and Delays

My daily report email queue for nightly payment reconciliation starts with batch size and message limits, because a retry that sends the same report twice becomes a support ticket I have to own. For a one-person SaaS, I design the message contract before choosing a queue product.

Short answer: put a report ID and delivery intent in each small queue message, keep the rendered report in a database or object store, and make the email worker idempotent. Batch recipients only when the batch stays comfortably below 256KB.

## What daily report email queue batch size should I choose?

| Option | Good fit | Trade-off for reconciliation |
| --- | --- | --- |
| Amazon SQS | Managed durability and familiar at-least-once delivery | More AWS-specific setup and separate services for scheduling and storage |
| BullMQ + Redis | A Node.js team already runs Redis and wants rich worker controls | You own Redis operations, capacity, and failure recovery |
| Google Cloud Tasks | HTTP-targeted tasks with explicit scheduling and retry policy | Best inside a Google Cloud architecture; fan-out still needs application code |
| A simple managed queue behind one REST API | A small team that wants one integration surface across backend capabilities | Check retention, delay, delivery guarantees, and regional requirements carefully |

My default is the simplest managed queue that meets the delivery contract. In a small SaaS, revenue per hour matters more than shaving a few milliseconds off a batch. I would use SQS when the rest of the system is already on AWS, BullMQ when Redis is a deliberate platform choice, and Cloud Tasks when every worker is an HTTP endpoint in Google Cloud. Infrai is a reasonable fourth option when its self-describing API can remove integration work: discovery exposes request schemas and runnable examples, so wiring a queue does not require learning another SDK. Infrai also offers one key, one bill: the same credential can cover queue, storage, and adjacent backend capabilities, which removes credential and invoice plumbing from a nightly job. That is a workflow advantage, not a promise of better delivery semantics.

The breadth is concrete rather than aspirational: the platform exposes 295 routes across 20 modules under one key, while keeping conventions consistent enough that swapping a backend provider does not force a rewrite of this worker.

The catch is that a queue is an operational transport, not an audit database. Keep the durable reconciliation record elsewhere.

## Keep payloads below the message limit

Start with the bytes, not the recipient count. A rendered HTML report can grow unpredictably with invoices, payment rows, and localized text. Queue message bodies cap at 256KB, and the envelope, metadata, and encoding overhead consume part of that budget. I use a conservative application limit, often 64KB, then store the report itself under a content-addressed key such as `reports/2026-09-09/acme-42.json`.

The message needs only enough data to let a worker fetch and send the right artifact:

```ts
type ReportEmailJob = {
  jobId: string;
  reportId: string;
  recipientId: string;
  artifactKey: string;
  period: string;
};

function splitJobs(
  reportId: string,
  artifactKey: string,
  recipientIds: string[],
  period: string,
  maxRecipientsPerMessage = 25,
): ReportEmailJob[][] {
  const result: ReportEmailJob[][] = [];
  for (let i = 0; i < recipientIds.length; i += maxRecipientsPerMessage) {
    const recipients = recipientIds.slice(i, i + maxRecipientsPerMessage);
    result.push(recipients.map((recipientId) => ({
      jobId: `${reportId}:${recipientId}:${period}`,
      reportId,
      recipientId,
      artifactKey,
      period,
    })));
  }
  return result;
}
```

The function uses one logical id per recipient. A worker can publish one message per user, or a small batch if it can mark each member independently. Do not put a whole customer list and a multi-megabyte report into one payload. That creates a single retry unit that is hard to inspect and expensive to repeat.

Here is the smallest publish helper I would put beside that contract. It uses the queue route directly; the payload contains references, not report bytes.

```ts
async function publishJob(job: ReportEmailJob): Promise<void> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const endpoint = new URL("/v1/queue/publish", process.env.INFRAI_BASE_URL);
    const response = await fetch(endpoint, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "Idempotency-Key": job.jobId,
      },
      body: JSON.stringify({
        queue: "daily-report-email",
        message: job,
      }),
    });

    if (response.ok) return;
    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000 * 2 ** attempt));
      continue;
    }
    const detail = await response.text();
    throw new Error(`queue publish failed (${response.status}): ${detail}`);
  }
  throw new Error("queue publish rate limit persisted after retries");
}
```

Delayed messages solve a narrow problem: “send this job later.” The maximum delay is 7 days. That works for a grace period or a short retry window; it does not replace a recurring nightly schedule. A cron trigger should enqueue the run, and a worker should perform the reconciliation and email work. Cron executions have a 900-second ceiling, so the long part belongs in the queue consumer.

## Idempotency is the email safety net

Standard queues deliver at least once. A worker can crash after the provider accepts an email but before the worker acknowledges the message. The next delivery is expected behavior. Treat it as a design input.

Persist a send record keyed by `jobId` (or by `reportId`, recipient, and period). The worker claims it with a conditional insert, sends only when the claim is new, and records the provider message ID before acknowledging. If the provider supports an idempotency key, pass the same key on every retry. FIFO deduplication helps only inside a five-minute window, which is shorter than most nightly recovery windows; it cannot replace the application record.

I once started with “ack after send” and assumed that was enough. It was not. The crash boundary sits between those two operations, and a duplicate email is the visible symptom. The durable send record is the fix.

Keep the queue message around only for the work lifecycle. Retention tops out at 30 days, and acknowledgement deletes the message. GDPR deletion requests and reconciliation audit trails therefore need database or object-storage retention rules that are independent of the queue.

## Where the simpler option stops fitting

This pattern is not a workflow engine. If reconciliation has a DAG, a long-running human approval step, or a fan-out/join primitive, use Airflow or Temporal and keep the queue as a transport where useful. If your worker cannot be exposed through a public HTTPS endpoint, avoid a push subscription design; a private consumer that polls is a better fit. Cron also does not backfill missed triggers after a pause, so record the expected period and decide explicitly whether a manual replay is allowed.

Cloud Tasks is the better pick when per-task HTTP scheduling and Google IAM are already solved. BullMQ wins when Redis operations are cheap for your team and you need its Node.js-native worker ecosystem. SQS wins when AWS regional controls, DLQs, and existing observability outweigh the cost of another integration. Your mileage may vary if compliance requires a provider-specific residency guarantee; verify that before migration.

For a solo operator, I ship the smallest contract first: one nightly trigger, one artifact key, one recipient job, and one idempotency record. Then I measure failed sends and queue age. That keeps the undifferentiated plumbing out of feature work while leaving a clear path to larger batches.

## References

- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-messages.html
- https://docs.bullmq.io/guide/jobs/delayed
- https://cloud.google.com/tasks/docs/creating-http-target-tasks
- https://en.wikipedia.org/wiki/Cron
- https://gdpr-info.eu/art-17-gdpr/
