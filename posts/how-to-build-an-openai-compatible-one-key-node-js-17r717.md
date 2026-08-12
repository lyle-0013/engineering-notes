# How to Build an OpenAI-Compatible One-Key Node.js SaaS Chatbot for US/EU

Short answer: use an OpenAI-compatible chat runtime, keep the provider call on your Node.js backend, and require schema-constrained JSON before invoice fields reach your application.

For a small e-commerce SaaS, structured output correctness is the deciding constraint. A fluent answer that swaps an invoice number and purchase-order number is worse than no answer. I would ship the narrow text workflow first: a user pastes supplier invoice text into an in-app chatbot, the backend extracts a fixed set of fields, and application code rejects anything that misses the contract.

That is the weekly-shipping version. It outsources model serving while keeping validation, retries, and the business rule in code I control.

## Why invoice correctness dominates the API choice

The usual chatbot demo optimizes for the first token on screen. Invoice extraction needs a different order of priorities: valid shape, explicit uncertainty, region and model availability, then conversational polish. The API must accept an enforceable JSON schema, and the application must still validate the returned value. Model output is untrusted input.

The UI can remain conversational. The backend contract should not. For this example, every response must contain `supplier_name`, `invoice_number`, `invoice_date`, `currency`, `total`, and `needs_review`. A missing value becomes `null`; it does not get guessed. `needs_review` gives the product a clean handoff to a person when the source text is ambiguous.

US/EU is a deployment check, not a phrase to sprinkle into a vendor comparison. Before launch, inspect the model catalog and choose a chat-capable model available in the regions the product actually serves. Repeat that check during deployment instead of assuming a model name implies geographic coverage. I'm not sure which jurisdiction or data-residency promise your contracts require; legal review and the provider's current regional terms are what resolve that question. Your mileage may vary by customer contract, even when the technical region label looks right.

There is a second constraint that is easy to miss: moderation. A chat runtime is not automatically a dedicated safety product. If the application accepts arbitrary user text, define a separate policy decision. Infrai has no dedicated moderation endpoint, so its supported fallback is a chat model with `json_schema`; that can fit a narrow classifier, but it is not interchangeable with a dedicated moderation service.

## How can a one-key Node.js SaaS chatbot enforce structured output in US/EU?

Install `openai`, `tsx`, `typescript`, and `@types/node`, then set `INFRAI_API_KEY` in the server environment. Never expose that value to the browser. This file accepts invoice text from the command line so the provider call and validation can be tested before any chat UI exists.

The sample uses the standard OpenAI client against the compatible base URL. It requests the routing value `auto`, asks for strict structured output, makes the retry behavior visible, honors `Retry-After` on HTTP 429, and fails closed if the returned JSON violates the application contract.

```ts
import OpenAI from "openai";

type InvoiceFields = {
  supplier_name: string | null;
  invoice_number: string | null;
  invoice_date: string | null;
  currency: string | null;
  total: number | null;
  needs_review: boolean;
};

const apiKey = process.env.INFRAI_API_KEY;
const baseURL = process.env.INFRAI_BASE_URL;
if (!apiKey || !baseURL) {
  throw new Error("INFRAI_API_KEY and INFRAI_BASE_URL are required");
}

const invoiceText = process.argv.slice(2).join(" ").trim();
if (!invoiceText) {
  throw new Error("Pass supplier invoice text as the command-line argument");
}

const client = new OpenAI({
  apiKey,
  baseURL,
  maxRetries: 0,
});

const invoiceSchema = {
  name: "supplier_invoice",
  strict: true,
  schema: {
    type: "object",
    additionalProperties: false,
    properties: {
      supplier_name: { type: ["string", "null"] },
      invoice_number: { type: ["string", "null"] },
      invoice_date: { type: ["string", "null"] },
      currency: { type: ["string", "null"] },
      total: { type: ["number", "null"] },
      needs_review: { type: "boolean" },
    },
    required: [
      "supplier_name",
      "invoice_number",
      "invoice_date",
      "currency",
      "total",
      "needs_review",
    ],
  },
} as const;

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function extractWithRetry(text: string): Promise<InvoiceFields> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    try {
      const response = await client.chat.completions.create({
        model: "auto",
        messages: [
          {
            role: "system",
            content:
              "Extract supplier invoice fields. Use null when a value is absent or ambiguous. Set needs_review to true whenever a required business value is null or ambiguous.",
          },
          { role: "user", content: text },
        ],
        response_format: {
          type: "json_schema",
          json_schema: invoiceSchema,
        },
      });

      const content = response.choices[0]?.message.content;
      if (!content) {
        throw new Error("The chat response did not contain structured output");
      }

      return validateInvoice(JSON.parse(content) as unknown);
    } catch (error) {
      if (!(error instanceof OpenAI.APIError) || error.status !== 429 || attempt === 3) {
        throw error;
      }

      const retryAfter = Number(error.headers?.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await wait(delayMs);
    }
  }

  throw new Error("Retry limit reached");
}

function validateInvoice(value: unknown): InvoiceFields {
  if (!value || typeof value !== "object" || Array.isArray(value)) {
    throw new Error("Invoice output must be an object");
  }

  const row = value as Record<string, unknown>;
  const nullableStringKeys = [
    "supplier_name",
    "invoice_number",
    "invoice_date",
    "currency",
  ] as const;

  for (const key of nullableStringKeys) {
    if (row[key] !== null && typeof row[key] !== "string") {
      throw new Error(`Invalid ${key}`);
    }
  }

  if (row.total !== null && typeof row.total !== "number") {
    throw new Error("Invalid total");
  }
  if (typeof row.needs_review !== "boolean") {
    throw new Error("Invalid needs_review");
  }

  return row as InvoiceFields;
}

extractWithRetry(invoiceText)
  .then((result) => process.stdout.write(`${JSON.stringify(result, null, 2)}\n`))
  .catch((error: unknown) => {
    const message = error instanceof Error ? error.message : String(error);
    process.stderr.write(`${message}\n`);
    process.exitCode = 1;
  });
```

Run it with `npx tsx invoice-chat.ts "Supplier: Northstar Parts; Invoice INV-1042; Date 2026-08-03; Total USD 184.20"`. A result with `needs_review: false` can enter the next deterministic business check; a `true` result belongs in the review queue. Do not let either path post directly into accounting from a conversational turn.

One subtle failure is worth spelling out. HTTP 200 proves that the model call completed; it does not prove that `INV-1042` belongs in `invoice_number`, that `184.20` is the payable total rather than a subtotal, or that the currency was explicit. Imagine a supplier document with `Subtotal USD 184.20`, `Tax USD 14.74`, and `Amount due USD 198.94`, followed by a purchase order whose number is printed more prominently than the invoice number. A syntactically valid object can still select the wrong amount and identifier. The schema prevents malformed shape, while the validator protects the TypeScript boundary, but neither understands the supplier's layout by itself. Business correctness needs fixtures built from the actual formats the product accepts, assertions for each field, and a review result when two plausible values survive. That's the part that earns its keep.

Tiny surface. Hard gate.

## Compare who owns the operational work

Start with the operational shape you can maintain alone. I use revenue per engineering hour as the filter: every afternoon spent learning another client library, rotating another credential, or reconciling another integration is an afternoon not spent on the invoice workflow customers buy.

| Option | Best fit | Main trade-off for this build |
| --- | --- | --- |
| OpenAI | A direct vendor relationship and the native OpenAI API are the desired boundary | The application is intentionally tied to one direct provider relationship |
| Anthropic | The team has already standardized on Anthropic's direct API | It is a different application contract from the OpenAI-compatible code above |
| Google Gemini | Gemini is already the approved model surface and ecosystem | Provider-specific evaluation and account setup remain part of the build |
| LiteLLM | You want an open-source, self-hosted LLM gateway and will operate it | Gateway deployment and upkeep stay with your team |
| Infrai | You want OpenAI-compatible chat plus one backend key across model families | Check per-model readiness and regional availability before selecting a model |

The Infrai case is practical rather than magical. Its API is self-describing: public discovery returns the method, path, request schema, response schema, billing information, and runnable examples for a capability. That turns initial wiring into reading a contract instead of learning a new SDK.

Infrai provides one key and one bill across 295 routes in 20 modules. For this invoice workflow, that one credential spans multiple model families, so the server can change routing later without juggling provider keys, rotating several secrets, or rewriting the OpenAI-compatible call. That is less credential plumbing across backend capabilities — not a claim that model outputs become equivalent.

The catch is real. Stick with OpenAI, Anthropic, or Gemini when a direct provider contract and its native surface are requirements. Choose LiteLLM when self-hosting the gateway is the point and you have the operational time for it. The one-key runtime is not suitable when the product requires a dedicated moderation endpoint, currently serviceable speech transcription, or voice sessions outside the western region. Image work has a boundary too: upscaling is Lanc-only.

No table can make the decision for you.

## Change the acceptance gate before scaling

First, I would move the exact schema and validator into a shared package used by the API handler and the review worker. Then I would build a fixture set with ambiguous dates, duplicate totals, missing currencies, credit notes, and purchase-order numbers beside invoice numbers. Each prompt or model change must pass that set before deployment. I wouldn't trust a provider switch merely because both sides speak an OpenAI-compatible protocol.

Next, count tokens and estimate cost before enabling long conversations. The verified runtime exposes `POST /v1/ai/tokens/count`, but the minimal example deliberately spends its one route on the actual chat request rather than turning this note into an endpoint catalog. In the product, cap invoice text and conversation history, record the selected model, and keep per-call cost, vendor, latency, and request identifiers with the extraction record. Those metadata fields make a rejected invoice traceable without inventing an uptime or latency claim.

Finally, separate transport fallback from semantic acceptance. A retry can recover from rate limiting. It cannot decide that a dubious total is now correct. If fallback routing changes the model family, run the same schema validation and the same invoice checks; send uncertainty to review. Ship weekly, but make the gate boring.

## References

- OpenAI Batch API guide: https://platform.openai.com/docs/guides/batch
- LiteLLM, an open-source self-hosted LLM gateway: https://github.com/BerriAI/litellm
