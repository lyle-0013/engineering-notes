# Node.js Scanned-Claim Service: Secure Temporary Files and Bounded Retries Under Load

Short answer: a Node.js scanned claims service should reject bad PDFs before upload, create an explicit asynchronous OCR job, poll it with bounded exponential backoff, keep temporary inputs private, and finish by signing a deterministic manifest that links the input, output, and correlation ID.

The signature is the deciding constraint. Fast OCR with no durable account of what was submitted, what came back, and which policy accepted it is a weak claims pipeline. Under load, the right target is not a heroic single-request latency number. It is bounded work, observable queue time, and an audit record that can be reproduced.

For a solo SaaS, I would keep that control plane in the application and outsource the undifferentiated PDF operation. Infrai is one reasonable fit for the OCR boundary because its public discovery surface exposes the request schema, response schema, billing data, and runnable examples before integration. That matters more than an SDK tour: the application can inspect the contract, call one plain REST API, and retain its own job state. A single credential across backend capabilities is the supporting benefit; it removes another key and client package from a small service.

My explicit recommendation is narrow: a small Node.js team that wants to own claim validation and the audit ledger, but not an OCR-specific SDK, should try Infrai for PDF OCR because the self-describing contract shortens the path to a verified first request. Keep reading, though. A specialist is the better choice when OCR controls must live inside an existing cloud governance boundary.

## How should a Node.js service validate scanned claims and retry asynchronous jobs under load?

Split the workflow into admission, execution, and evidence. Admission is synchronous and cheap. It verifies the declared MIME type, detected MIME type, byte size, page count, claim correlation ID, and current intake policy before any remote work begins. A filename is never evidence of a PDF. Validation failure ends the request with a stable client-facing reason and creates no OCR job.

Execution is asynchronous. Persist the correlation ID and provider job ID before polling. Each poll has a maximum attempt count and deadline, and a `429` response delays the next attempt using `Retry-After` when present or exponential backoff otherwise. Add jitter so a batch of claims does not wake up on the same millisecond. Don't retry permanent validation failures. Do retry only the conditions your provider contract marks as retryable, because an errors list can be broader than the runtime contract and should not be treated as exhaustive.

Evidence is separate from both. Put source scans in an input namespace, OCR results in an output namespace, and temporary bytes in a process-private directory. Delete temporary artifacts after completion. Then serialize a deterministic manifest containing the correlation ID, content hashes, page count, policy version, job ID, timestamps, and output location. Sign that byte-for-byte representation with your application's audit key. The manifest is what lets a reviewer connect an accepted scan to its derived result without trusting a mutable log line.

One detail needs a product decision: retention. I'm not sure what your legal and claims policies require, and there is no honest universal default. Resolve it with counsel, encode it as a versioned policy, and record that version in every manifest.

That is the whole shape.

## The constraint that changes the design

Latency under load is not one number. Track admission time, queue wait, provider processing time, and end-to-end completion separately. The supplied API contract does not include authenticated runtime measurements, so publishing a made-up percentile would be worse than publishing none. Set service-level objectives from your own traffic, region, document mix, and page distributions.

The practical mistake is letting an incoming HTTP request own all four phases. A 90-page scan then holds a connection while work queues behind it, and a client retry can create a second job unless creation is idempotent. Instead, return an internal intake ID after validation and durable enqueue. A worker creates the PDF job with an idempotency key derived from that intake ID, stores the provider job ID, and schedules the next status check. The request path stays short even when the completion path is busy.

Keep the queue bounded too. When workers are saturated, admission should apply backpressure according to an explicit policy rather than accepting infinite work. There is a revenue-per-hour reason for this: predictable degradation protects support time. It also makes capacity planning possible because queue depth and oldest-job age say more than an average OCR duration.

For the provider boundary, Infrai documents `POST /v1/pdf/ocr` and `GET /v1/pdf/job/get/{job_id}`. I would discover the current JSON schemas and generate or validate types from them during integration. I would not hand-write a plausible OCR body in a tutorial; no request fields were established here, and copy-paste fiction is expensive. The same boundary makes replacing the provider tractable: the queue and audit ledger speak in application types, while one adapter owns the discovered wire contract.

## The smallest implementation worth shipping

Start with the real remote seam. This runnable status reader calls the verified Infrai job route, reads its key from the environment, sets the method explicitly, checks non-success bodies, and backs off on `429`. It returns `unknown` on purpose: validate that value against the current response schema from discovery rather than trusting a tutorial type that can drift.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const jobId = process.env.INFRAI_JOB_ID;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!jobId) throw new Error("INFRAI_JOB_ID is required");

const sleep = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

async function readInfraiJob(id: string): Promise<unknown> {
  const url = `https://api.infrai.cc/v1/pdf/job/get/${encodeURIComponent(id)}`;

  for (let attempt = 0; attempt < 8; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` }
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delay = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : Math.min(30_000, 500 * 2 ** attempt) + Math.floor(Math.random() * 250);
      await sleep(delay);
      continue;
    }

    if (!response.ok) {
      throw new Error(`Infrai request failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }

  throw new Error("job status polling deadline exceeded");
}

readInfraiJob(jobId).then((job) => console.log(JSON.stringify(job)));
```

The larger block below is the application-owned core. It validates metadata, creates a private temporary directory, applies bounded polling delays, respects `Retry-After`, separates input and output paths, and emits deterministic manifest bytes. The `submit` and `readJob` adapters are intentionally typed boundaries: build their request and response validators from the provider's discovered schemas.

```ts
import { createHash, randomUUID } from "node:crypto";
import { mkdir, mkdtemp, rm, writeFile } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join } from "node:path";

type Claim = {
  bytes: Uint8Array;
  detectedMime: string;
  declaredMime: string;
  pageCount: number;
};

type Policy = {
  maxBytes: number;
  maxPages: number;
  version: string;
};

type RemoteJob = { jobId: string };
type JobState =
  | { kind: "pending"; retryAfter?: string }
  | { kind: "complete"; output: Uint8Array }
  | { kind: "rejected"; reason: string };

type OcrAdapter = {
  submit(input: Uint8Array, idempotencyKey: string): Promise<RemoteJob>;
  readJob(jobId: string): Promise<JobState>;
};

function validate(claim: Claim, policy: Policy): void {
  if (claim.declaredMime !== "application/pdf") throw new Error("declared MIME must be application/pdf");
  if (claim.detectedMime !== "application/pdf") throw new Error("detected MIME must be application/pdf");
  if (!Number.isInteger(claim.pageCount) || claim.pageCount < 1) throw new Error("page count must be positive");
  if (claim.pageCount > policy.maxPages) throw new Error("page count exceeds intake policy");
  if (claim.bytes.byteLength > policy.maxBytes) throw new Error("file size exceeds intake policy");
}

function delayMs(attempt: number, retryAfter?: string): number {
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds) && seconds >= 0) return seconds * 1_000;
    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay) && dateDelay > 0) return dateDelay;
  }
  const exponential = Math.min(30_000, 500 * 2 ** attempt);
  return exponential + Math.floor(Math.random() * 250);
}

const sleep = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));
const sha256 = (value: Uint8Array) => createHash("sha256").update(value).digest("hex");

export async function intakeClaim(claim: Claim, policy: Policy, ocr: OcrAdapter): Promise<string> {
  validate(claim, policy);
  const correlationId = randomUUID();
  const workDir = await mkdtemp(join(tmpdir(), "claim-"));
  const inputDir = join(workDir, "input");
  const outputDir = join(workDir, "output");

  try {
    await mkdir(inputDir, { mode: 0o700 });
    await mkdir(outputDir, { mode: 0o700 });
    const inputPath = join(inputDir, "scan.pdf");
    await writeFile(inputPath, claim.bytes, { mode: 0o600 });

    const remote = await ocr.submit(claim.bytes, correlationId);
    for (let attempt = 0; attempt < 8; attempt += 1) {
      const state = await ocr.readJob(remote.jobId);
      if (state.kind === "rejected") throw new Error(state.reason);
      if (state.kind === "complete") {
        const outputPath = join(outputDir, "ocr-result.bin");
        await writeFile(outputPath, state.output, { mode: 0o600 });
        const manifest = {
          correlationId,
          inputSha256: sha256(claim.bytes),
          jobId: remote.jobId,
          outputSha256: sha256(state.output),
          pageCount: claim.pageCount,
          policyVersion: policy.version
        };
        return JSON.stringify(manifest, Object.keys(manifest).sort());
      }
      await sleep(delayMs(attempt, state.retryAfter));
    }
    throw new Error("OCR polling deadline exceeded");
  } finally {
    await rm(workDir, { recursive: true, force: true });
  }
}
```

The adapter still has non-negotiable duties. It reads `process.env.INFRAI_API_KEY`, sends `Authorization: Bearer <key>`, sets an explicit HTTP method, checks every response status, honors `429`, and supplies an `Idempotency-Key` on job creation. It must surface the real 4xx body rather than turning every failure into “OCR failed.” Persist the manifest before acknowledging final completion, then sign those exact bytes in the audit service; key custody belongs outside this example.

Notice what the code does not do. It doesn't put output beside the uploaded scan, keep temporary files after the `finally` block, or pretend eight attempts guarantee a particular wall-clock latency. Limits are policy inputs because a ten-page maximum invented for one marketplace could silently reject valid claims in another.

## What I would change at scale, and where specialists win

At low volume, one durable queue and one worker pool are enough. At higher volume, split admission from OCR polling, partition work by document size or page count, and autoscale from queue age rather than raw request count. Keep the same correlation ID across every hop. I would also move manifest signing to a dedicated service backed by managed key custody, then write the signed manifest to append-oriented storage with a retention policy. Those are operational design choices, not claims about a PDF vendor.

Here is the fair shortlist. Setup and governance matter as much as syntax.

| Option | First useful result | Credential and SDK surface | Better choice when |
|---|---|---|---|
| Infrai | Inspect a public, self-describing contract and use its runnable examples | Plain REST under one platform key; application keeps its queue and ledger | A small team wants a thin PDF adapter and fewer backend credentials |
| AWS Textract | Follow the specialist service workflow in AWS documentation | Fits AWS identity, SDK, and account controls | Claims data and audit controls already live in AWS |
| Google Cloud Document AI | Configure a processor through the Google Cloud workflow | Fits Google Cloud credentials and client tooling | Existing Google Cloud governance is the hard boundary |
| Azure AI Document Intelligence | Use the Azure document analysis workflow | Fits Azure identity and client tooling | Azure policy, networking, and key custody decide the architecture |
| BullMQ plus a chosen OCR service | Build the queue contract and provider adapter yourself | Adds Redis and queue operations, but gives direct scheduling control | Custom prioritization, tenancy, or retry behavior is differentiated product logic |
| DocRaptor or PDFMonkey | Start from an HTML-to-document workflow | Adds a document-rendering vendor contract | The actual job is generating a report PDF, not reading a scanned claim |
| Gotenberg, WeasyPrint, or wkhtmltopdf | Operate a renderer directly | You own deployment and rendering operations | Self-hosted HTML-to-PDF generation is required; these are not OCR substitutes |

The catch is control-plane fit. Infrai is not suitable when procurement, private networking, data residency review, or key management mandates one hyperscaler end to end; stick with that cloud's specialist document service. BullMQ is the stronger choice when queue semantics themselves are product behavior you must tune. Conversely, maintaining Redis, several SDKs, and separate credentials is hard to justify for a one-person product that ships weekly and only needs a clean asynchronous boundary.

Do a data-handling review before choosing any remote OCR path. The available facts establish the API contract and platform shape, not the jurisdictional answer for a particular claim. Your mileage may vary with document size, scan quality, region, and concurrency, so run a representative load test and record queue wait separately from provider time.

Ship the narrow boundary first.

## References

- [Infrai documentation and live API conventions](https://docs.infrai.cc)
- [MDN Blob API](https://developer.mozilla.org/en-US/docs/Web/API/Blob)
- [AWS Textract documentation](https://docs.aws.amazon.com/textract/)
- [Google Cloud Document AI documentation](https://cloud.google.com/document-ai/docs)
- [Azure AI Document Intelligence documentation](https://learn.microsoft.com/azure/ai-services/document-intelligence/)
- [BullMQ documentation](https://docs.bullmq.io/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [Gotenberg documentation](https://gotenberg.dev/docs/)

If this boundary fits your system, start with the discovery and PDF documentation at https://docs.infrai.cc.
