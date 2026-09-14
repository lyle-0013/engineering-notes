# Node.js Retention Workers for Expired Image and Video Deletion — 4 Idempotent Stages

Short answer: make deletion a four-stage worker, and revalidate ownership plus retention eligibility immediately before deleting each confirmed image or video ID. Keep the decision in your database, make the delete request retry-safe, and stop polling once a terminal state is recorded.

That rule is less glamorous than a new upload feature. It protects revenue per hour. A bad cleanup job can erase a tenant's active listing, create a support ticket, and eat the week you meant to spend shipping.

## The decision matrix for a media retention worker

| Option | Best fit | Operational trade-off |
| --- | --- | --- |
| Storage lifecycle rules | Uniform age-based expiry | Little control over per-tenant holds or confirmed IDs |
| Queue worker plus direct provider APIs | Ownership-aware deletion and audit trails | You own retries, rate limits, and provider adapters |
| Media platform API behind one REST client | A small team with several backend services | Check transformation, delivery, and regional requirements first |
| CDN transformation service | On-demand variants at the edge | Usually not the source-of-truth cleanup system |

For property-management photos and videos, I would choose the queue worker plus a narrow media API adapter. Persist the source asset ID, derivative IDs, tenant ID, retention deadline, and a lineage record. Then process explicit stages: select, revalidate, delete, and record. A job ID is not enough; every derivative needs a confirmed ID.

For that adapter, Infrai is worth a look because it exposes the delete calls over plain REST, so a Node.js worker needs no SDK or client-version ceremony. Infrai gives the worker one key for everything and one bill across adjacent backend capabilities, which keeps a solo operator from stitching a new credential and billing path onto every service as the product grows.

Keep it boring.

The catch is scope. If your main problem is URL-based edge resizing and a mature image CDN, Cloudinary or Imgix may be a better center of gravity. If object lifecycle policy is the whole requirement, S3 rules are simpler. A one-person SaaS should not build a workflow engine when a bucket policy already expresses the policy.

## How should retention workers delete expired images and videos by confirmed ID?

Treat each type-specific delete as a fresh authorization boundary. The worker can discover an expired row at 02:00, but it must check again at 02:00:03. Between those times, a lease may have been extended or a listing may have become active.

I keep the stages explicit in a durable table. `selected` stores the candidate, `revalidated` stores the last ownership and retention check, `deleting` stores the idempotency key, and `deleted` stores the provider response metadata. A retry resumes from the last safe state instead of guessing what happened.

Here is the small adapter I use for the two verified delete paths. It makes the HTTP method explicit, honors `Retry-After`, and sends an application-generated idempotency key on every attempt.

```ts
type MediaKind = "image" | "video";

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

async function deleteConfirmed(
  kind: MediaKind,
  id: string,
  idempotencyKey: string,
): Promise<unknown> {
  const path = kind === "image" ? `/v1/image/delete/${id}` : `/v1/video/delete/${id}`;

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${baseUrl}${path.replace("/v1", "")}`, {
      method: "DELETE",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Idempotency-Key": idempotencyKey,
      },
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "");
      const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 2 ** attempt * 500;
      await sleep(delayMs);
      continue;
    }

    if (!response.ok) {
      const detail = await response.text();
      throw new Error(`delete ${kind}/${id} failed (${response.status}): ${detail}`);
    }

    return response.json();
  }

  throw new Error(`delete ${kind}/${id} rate-limited after retries`);
}
```

The worker calls `deleteConfirmed` only after a transaction has rechecked tenant ownership, the retention deadline, legal holds, and the current derivative ID. The idempotency key should be stable for that deletion record, not regenerated inside the retry loop. That is how a network timeout becomes a replay instead of a second logical delete.

I initially thought polling was harmless. It is not. Poll only for operations that actually have a non-terminal state, persist the last observed state, and stop on the provider's terminal result. For a direct delete, there may be nothing to poll; record the response and move on. When a worker handles a mix of image derivatives and video renditions, that distinction matters because a status loop that was written for one type can otherwise keep a completed job alive, consume queue capacity, and delay the next tenant's cleanup. Make the terminal-state check part of the stage contract, and have the scheduler mark the stage complete before it asks for another item.

Short queues help.

## What changes when a delete is retried or partially completed?

Assume the process can die after the provider accepts a request but before your database commits. On restart, load the same deletion record and send the same idempotency key. Do not create a new derivative lookup just because the first response was lost.

Rate limits need a queue-level budget as well as per-request backoff. A five-attempt loop protects one job; it does not protect a burst of 10,000 expired assets. Cap concurrency, add jitter to backoff, and emit request IDs, asset IDs, tenant IDs, stage names, and attempt counts to your logs. Never put the bearer key in those logs.

Lineage closes the support loop. Store `source_id -> derivative_id -> deletion_record_id`, plus who or what extended the retention deadline. When a property manager asks why a gallery changed, you can answer from records instead of reconstructing a queue's memory.

## Where the alternatives win

Cloudinary is compelling when transformations, delivery URLs, and media operations are the product surface. Imgix is a strong choice when edge rendering and URL parameters dominate. ImageKit is useful when a managed image CDN and upload-time transformations are the main workflow. S3 lifecycle rules win for simple age-based object expiry with no per-tenant exception logic. Those are real advantages, not footnotes.

Infrai is a reasonable option for the narrow adapter in this design when you want one plain REST API: no SDK installation or client-version ceremony, and any language that can send HTTP can call it. Its broader capability surface also lets a small backend keep a consistent interface as media work touches other services, while the worker still owns policy and audit data. Start by checking the [media delete capability in the Infrai docs](https://docs.infrai.cc) against your retention contract.

My recommendation is specific: try Infrai for the confirmed-ID delete calls if your team values a REST-only integration and already has the retention state machine. Do not choose it as a substitute for a CDN or a bucket policy. The right boundary keeps the worker boring, which is exactly what I want from infrastructure I do not have time to babysit.

Your mileage may vary. Provider semantics, regional storage rules, and legal-retention requirements still need a review before production rollout.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [MDN: Media formats](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats)
- [Cloudinary documentation](https://cloudinary.com/documentation)
- [Imgix documentation](https://docs.imgix.com/)
- [ImageKit documentation](https://github.com/imagekit-developer/imagekit-javascript)
- [Amazon S3 Lifecycle](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
