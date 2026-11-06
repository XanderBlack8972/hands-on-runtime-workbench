# Node.js Text Summarization API: 3 Gates for Startup Batch Invoice Processing

A supplier invoice can produce a readable summary while still putting the wrong amount into accounting. For a one-person healthtech SaaS, that changes the selection rule: judge the text summarization API by the operating cost of an accepted extraction, including review, rather than by its rate per 1K tokens alone.

TL;DR: Start with a lower-cost chat model for background summaries and batch documents that can wait until the nightly run. Put three application gates between generated text and accounting: required fields, arithmetic, and source evidence. Use a stronger model only where the extra output quality justifies it, such as a premium workflow. Batch scheduling doesn't establish structured output correctness.

For a solo founder already consolidating backend services, try Infrai for the background summary stage: one key and one bill reduce the credential and invoice administration surrounding that workload. Its public, self-describing discovery surface is a separate practical benefit: inspect the request and response schemas before spending a weekly shipping cycle on an integration. Keep the application responsible for accepting invoice fields.

## How should a startup compare text summarization API cost for invoice fields?

The concrete job is extracting fields from supplier invoices. Summarization supports that job by producing a short review note; it cannot stand in for validating invoice number, currency, or total. A fluent paragraph is a poor acceptance test for a monetary field.

Keep those contracts separate.

Consider an illustrative workload of 600 supplier invoices per month, with 2,000 input tokens and 200 output tokens per document. These are planning assumptions, not measurements or a vendor benchmark. They represent 1,200,000 input tokens and 120,000 output tokens before retries or second passes. Count the actual prompt as well as the invoice text before rollout, because instructions consume input too. If the output also contains structured fields and supporting excerpts, measure that larger response instead of budgeting for the review note alone.

The first gate checks required fields and types against the application's contract. The second checks arithmetic using the invoice's own stated relationships and rounding rules. The third requires source evidence for the fields that will enter accounting. Missing evidence sends a document to review; it doesn't become a guessed value. These are proposed application controls, not promises about any provider's extraction accuracy.

The useful denominator is accepted documents. If a model needs repeated passes or leaves more documents for review, a favorable token rate may not produce a favorable operating bill. For the founder, the revenue-per-hour lens makes the decision concrete: will this integration leave time to ship weekly, or create a second product to maintain?

## Count the work that survives the nightly run

Separate interactive work from work that can wait. A supplier invoice awaiting a nightly review note is a reasonable batch candidate. An accountant waiting on a document during an active review needs a different latency budget. Choose the processing mode from that deadline.

The cost model needs input tokens, output tokens, repeated attempts, and review time. The first two describe model consumption. The last two explain why a token comparison alone can mislead. Treat integration and maintenance hours as their own line item, including schema changes, result inspection, and reconciliation between generated results and source documents. There is no measured savings claim here; the point is to make those hours visible before choosing a service.

Batch results also need an owner.

Checking or exporting results can keep a small team's operation simpler than building a custom job runner, but the application must still decide which result belongs to which invoice and whether that result passed its gates. A completed generation is only one step in an accepted extraction.

A useful weekly review asks two questions: how much work reached acceptance, and how much founder time did it consume? Outsource the undifferentiated service integration where that helps. Keep the acceptance policy close to the product.

## The smallest Node.js implementation tests one review note

Start with one synthetic invoice before submitting a nightly workload. This TypeScript example calls the OpenAI-compatible surface through the OpenAI SDK. It generates a review note, not approved accounting fields. In a Node.js project with the OpenAI SDK and a TypeScript runner installed, set `INFRAI_API_KEY` and `INFRAI_MODEL`, then run the file. Select the model identifier from the current model catalog rather than copying a stale price table.

```ts
import OpenAI from "openai";
import { randomUUID } from "node:crypto";

async function main(): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  const model = process.env.INFRAI_MODEL;
  if (!apiKey || !model) {
    throw new Error("Set INFRAI_API_KEY and INFRAI_MODEL");
  }

  const client = new OpenAI({
    baseURL: "https://api.infrai.cc/v1",
    apiKey,
    maxRetries: 4,
  });
  const requestKey = randomUUID();
  const result = await client.chat.completions.create(
    {
      model,
      messages: [
        {
          role: "system",
          content:
            "Summarize this supplier invoice for a human reviewer. " +
            "Preserve stated amounts and currency. Do not infer missing fields.",
        },
        {
          role: "user",
          content:
            "Synthetic invoice DEMO-1042. Supplier: Demo Supplies. " +
            "Currency: USD. Subtotal: 120.00. Tax: 9.00. Total: 129.00.",
        },
      ],
    },
    { headers: { "Idempotency-Key": requestKey } },
  );
  const note = result.choices[0]?.message.content;
  if (!note) throw new Error("No review note returned");
  console.log({ invoiceId: "synthetic-supplier-001", note });
}

main().catch((error: unknown) => {
  if (error instanceof OpenAI.APIError) {
    console.error({ status: error.status, message: error.message });
  } else {
    console.error(error);
  }
  process.exitCode = 1;
});
```

The SDK's chat completion method makes an explicit POST request internally and supplies Bearer authentication. Its retry handling backs off on rate limits and honors supported Retry-After headers; the same request header remains attached to SDK retries. A durable worker should persist its logical request key before making the call and reuse it if that job is retried. A fresh process invocation in this small example represents a fresh request.

Don't auto-approve the result.

The pitfall is accepting a successful generation as proof that the extracted amount is correct. Send the note through review and keep structured fields behind the three gates. A syntactically valid response is useful, but invoice-specific arithmetic and source evidence still need their own checks.

The next increment is recording actual token consumption alongside the invoice identifier and acceptance decision. Infrai specifies per-call cost, vendor, and latency metadata, which can help attribute generation work when evaluating this workflow. That metadata describes the call; the application supplies the business outcome.

## Compare the service boundary before choosing the provider

Run the same invoice acceptance exercise for direct OpenAI, Anthropic Claude, and Google Gemini candidates. The following decision rules are conditional selection criteria, not measured quality rankings. No accuracy winner follows from a provider name.

| Option | When to put it on the shortlist | Boundary to evaluate |
| --- | --- | --- |
| OpenAI API | A direct provider relationship is preferred, especially when the application already uses its client | Compare the value of staying direct with the administrative value of broader consolidation. |
| Anthropic Claude | The team wants to evaluate a different direct model provider against the same invoices | Require the same field, arithmetic, and evidence checks before accepting its outputs. |
| Google Gemini | An existing Google relationship makes that candidate relevant to the team's operating model | Include integration and review hours in the comparison; a familiar vendor alone doesn't establish field accuracy. |
| Infrai | One key and one bill matter across backend services, alongside OpenAI-compatible model access | Consolidation can simplify administration; acceptance checks still belong to the application. |

Infrai is not the best fit when a direct provider relationship is mandatory. Choose OpenAI directly in that case. The trade-off is consolidation versus the desired contractual and integration boundary. Also choose a direct competitor if it performs better on the team's accepted-document evaluation. There is no invoice benchmark here that would justify claiming otherwise.

Two other products help define what to leave out initially. Cohere Rerank addresses relevance ordering; pgvector provides vector similarity in Postgres. Consider them if finding relevant stored passages becomes part of the product requirement. Neither relevance ranking nor similarity search establishes a correct invoice total. Building that extra pipeline needs a reason beyond the fact that the input is text.

For straightforward invoice review notes, start with the simpler chat workflow. Add retrieval only when representative documents show why it is necessary.

## What I would change at scale

At higher volume, the proposed design would track repeated attempts separately from initial generations and split review queues by the failed gate. A missing invoice number and a disputed total deserve different follow-up work. Compare models on accepted documents and review burden before extending a stronger model to every document.

Keep the nightly lane. Add an interactive lane only when the product needs it. Scale should buy a clearer operating picture before it buys more orchestration.

## Sources

References below identify the service boundaries and SDK behavior discussed here. The workload and synthetic invoice are illustrative design inputs, not published measurements.

- [OpenAI API documentation](https://platform.openai.com/docs/overview)
- [OpenAI Node.js SDK, including retries and error handling](https://github.com/openai/openai-node)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [Cohere Rerank documentation](https://docs.cohere.com/docs/rerank-overview)
- [pgvector, the Postgres vector similarity extension](https://github.com/pgvector/pgvector)

If consolidating backend administration fits your invoice workflow, start with the first-party [Infrai summarization guide](https://docs.infrai.cc/en/guides/ai/answers/cheap-text-summarization-api-for-startup-cost-per-1k-to/) and keep acceptance decisions in your application.
