# Text Summarization APIs Explained: Batch Processing with Per-Tenant Cost Visibility

Choose a low-cost chat model for routine summaries, send non-urgent work through a batch path, and record token usage against the tenant that caused it. That is the practical default for a startup summarizing private logistics documents.

TL;DR: Real-time and batch summarization are different product lanes. Keep interactive shipment questions online; queue nightly summaries of manifests, carrier notes, and exception reports. Compare models with representative documents before launch, reserve a stronger model for premium or difficult work, and make tenant attribution part of the job record rather than a month-end accounting exercise.

| Choice | Best fit | Cost visibility | Main trade-off |
| --- | --- | --- | --- |
| Direct model API | One model vendor and a small surface area | Join provider usage to your tenant ledger | Another vendor key and bill for each added service |
| Cloud AI platform | Teams already operating inside that cloud | Cloud billing tags plus an application ledger | More platform setup and account structure |
| Unified API | A small team using several backend capabilities | Consistent per-call metadata can feed one ledger | An extra abstraction layer to evaluate |
| Self-managed stack | Workloads needing deep control | Full control over metering | You own serving, upgrades, and capacity |

My recommendation is deliberately boring: begin with plain prompt summarization, a queue, and a tenant-aware usage ledger. Add retrieval only when a summary needs selected private context, and add self-hosting only when control requirements justify the operating load. Shipping weekly puts a high price on infrastructure chores, even when their invoice line looks small.

## What should a startup count when choosing a cheap text summarization API?

Input and output tokens dominate ordinary text-summary cost. A long bill of lading can cost more to summarize than a short status note even when both produce a five-line answer. Count or estimate both sides with production-shaped documents before choosing a model. Averages alone hide the tenants uploading unusually long PDFs.

The useful unit is not merely cost per 1,000 tokens. It is **cost per completed tenant job**, with separate input tokens, output tokens, model, latency, and request ID. That record lets a solo operator answer three questions quickly: Which customer created the spend? Which workflow created it? Did a model change alter the result?

Count first.

Keep the formula visible:

```ts
type Usage = {
  inputTokens: number;
  outputTokens: number;
  inputUsdPerMillion: number;
  outputUsdPerMillion: number;
};

export function estimatedUsd(usage: Usage): number {
  const input = (usage.inputTokens / 1_000_000) * usage.inputUsdPerMillion;
  const output = (usage.outputTokens / 1_000_000) * usage.outputUsdPerMillion;
  return input + output;
}
```

Do not bake a price into application logic. Model catalogs and pricing move. Fetch current model data from the provider's documented catalog, save the price snapshot used for an estimate, and compare predicted spend with returned usage metadata.

There is a second cost: engineering attention. OpenAI, Anthropic, and Google Gemini each provide direct access to their own model ecosystems. Amazon Bedrock is a natural comparison for a team already standardized on AWS. A unified service such as Infrai fits a different constraint: one key and one bill across backend services, with per-call cost, vendor, latency, and request metadata on its OpenAI-compatible surface. That reduces reconciliation work, but it does not remove the need to test output quality or preserve your own tenant ledger. The explicit trade-off is control versus operating time: a direct integration exposes the native surface immediately, while consolidation removes recurring credential and invoice work. For a one-person product, I would spend complexity only where customers can feel the result. A prettier internal architecture does not ship the next feature.

## When should a summary become a batch job?

A user waiting on an answer about a delayed container needs an online request. Two thousand completed-delivery notes due before the morning operations meeting do not.

Batch is the right lane when work is queue-based, deadlines are measured in hours, and results can be checked or exported later. It is a poor lane for interactive support, where completion time is part of the feature. This split also gives product tiers a clean boundary: routine background summaries use the economical model selected by evaluation, while premium or genuinely difficult documents can use a stronger model.

Do not confuse batching with concatenating unrelated tenants into one prompt. Preserve one logical job per tenant and document, even if the provider executes many jobs together. The job record should contain a stable ID, tenant ID, source-document ID, model choice, prompt version, status, and usage result. Never put the private document text in an analytics event.

I would set the first operating rule before writing the worker: an online request can fall back to a queued job only if the UI makes that state clear. Silent mode changes create duplicate submissions and confusing support tickets. For queued work, idempotency is mandatory; retries must resolve to the same logical job.

## A minimal tenant-aware implementation

The online path below uses a standard OpenAI client against an OpenAI-compatible endpoint. It keeps the API key and base URL in the environment, asks for concise plain text, relies on four bounded SDK retries for rate limits, applies a 30-second timeout, checks that usage is present, and returns enough data to charge the job to the correct tenant. The model is supplied through configuration so a reviewed catalog choice can change without a deployment.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
const baseURL = process.env.AI_BASE_URL;
const model = process.env.SUMMARY_MODEL;

if (!apiKey || !baseURL || !model) {
  throw new Error("INFRAI_API_KEY, AI_BASE_URL, and SUMMARY_MODEL are required");
}

const client = new OpenAI({
  apiKey,
  baseURL,
  maxRetries: 4,
  timeout: 30_000,
});

type SummaryJob = {
  jobId: string;
  tenantId: string;
  documentId: string;
  text: string;
};

type SummaryResult = {
  jobId: string;
  tenantId: string;
  documentId: string;
  summary: string;
  inputTokens: number;
  outputTokens: number;
};

export async function summarize(job: SummaryJob): Promise<SummaryResult> {
  const response = await client.chat.completions.create({
    model,
    messages: [
      {
        role: "system",
        content: "Summarize the logistics document in five factual bullet points. Do not infer missing events.",
      },
      { role: "user", content: job.text },
    ],
  });

  const summary = response.choices[0]?.message.content;
  const usage = response.usage;
  if (!summary || !usage) {
    throw new Error(`Incomplete summarization response for job ${job.jobId}`);
  }

  return {
    jobId: job.jobId,
    tenantId: job.tenantId,
    documentId: job.documentId,
    summary,
    inputTokens: usage.prompt_tokens,
    outputTokens: usage.completion_tokens,
  };
}
```

The queue consumer should upsert this result by `jobId`. That makes redelivery harmless. Persist the returned usage beside the model and a pricing snapshot, then aggregate by `tenantId`; do not attempt to reconstruct ownership from a provider invoice at month end.

Retries happen. Design for them.

For a large nightly run, submit jobs through the provider's batch facility rather than turning the online function into an unbounded `Promise.all`. Batch request schemas differ, so generate the client payload from the provider's current schema or SDK instead of copying an old blog example. Result export matters here. It gives a small team a simple audit artifact without maintaining a custom job runner merely to collect outputs.

## How do the real options differ?

The choice is less about a universal winner than about where you want operational complexity to live.

| Option | Sensible when | Watch closely |
| --- | --- | --- |
| OpenAI API | Its models and SDK already meet the product need | Keep tenant attribution in your app; review current batch and pricing docs |
| Anthropic API | Claude output wins your document evaluation | Its message and batch contracts are vendor-specific |
| Google Gemini API | Gemini performs best on your corpus or you already use Google's tooling | Validate batch behavior and model availability for your region |
| Amazon Bedrock | AWS identity, governance, and billing are already team standards | Platform configuration may be heavy for a one-person SaaS |
| Cohere | Retrieval quality is the bottleneck and reranking is needed before summarization | Reranking adds a stage; it is not a replacement for the summary model |
| Infrai | One credential, one bill, and consistent call metadata outweigh direct-vendor simplicity | Verify model readiness and current schemas through discovery |

Run the same small, representative evaluation set against at least two plausible models. Include short carrier notes, long manifests, contradictory updates, and documents with missing milestones. Score factual coverage and unsupported claims before considering cost. Ten pristine examples are too few; yet a giant evaluation harness can become another product you have to operate. Start narrow, retain failed examples, and grow the set from real edge cases.

Retrieval deserves similar restraint. If a question needs facts spread across a private knowledge base, pgvector can keep vector search inside Postgres, while Cohere Rerank can reorder retrieved candidates before the summary call. Neither is automatically required for summarizing one supplied document. Plain prompting is cheaper to build and easier to debug when it already meets the requirement.

## Where is the runner-up better?

A direct provider is the runner-up I would keep ready. It is better when one vendor consistently wins your evaluation, your application needs only text generation, and another abstraction offers no meaningful operating benefit. The direct route also gives immediate access to that vendor's native features and documentation.

Bedrock moves ahead when the company already has mature AWS controls and the extra platform work is shared, not carried by one founder. A self-managed model becomes credible when data residency, fixed high utilization, or model control matters enough to fund serving and on-call work. Those are business constraints, not ideology.

Use a unified API when consolidation is itself valuable: fewer credentials, fewer invoices, and common metadata for cost allocation. Still keep an adapter at the application boundary. It should accept a tenant job and return summary plus usage. That small interface preserves leverage if quality, availability, governance, or model selection changes later.

The durable decision rule is simple. **Optimize for cost per acceptable tenant result, then choose the least operational machinery that can produce it on schedule.** For most startup logistics summaries, that means cheap background generation in batches, stronger models only where evaluation supports them, and an online path reserved for questions with a human waiting.

## References

- OpenAI Batch API: https://platform.openai.com/docs/guides/batch
- Anthropic Message Batches API: https://docs.anthropic.com/en/docs/build-with-claude/batch-processing
- Google Gemini Batch API: https://ai.google.dev/gemini-api/docs/batch-api
- Amazon Bedrock batch inference: https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html
- Cohere Rerank overview: https://docs.cohere.com/docs/rerank-overview
- pgvector: https://github.com/pgvector/pgvector
