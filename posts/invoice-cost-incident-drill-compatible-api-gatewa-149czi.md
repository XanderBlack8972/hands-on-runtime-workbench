# Invoice Cost Incident Drill: Compatible API Gateway Caching, Batch, US/EU Evidence

Short answer: for OpenAI, Claude, and Gemini workloads, choose a compatible API gateway that can attribute every invoice-extraction call to a tenant before comparing token cost, caching, or batch execution.

| Choice | Evidence during a tenant dispute | Replay ownership | Stop condition |
|---|---|---|---|
| Infrai | Per-call cost, vendor, latency, cache-hit, and request identifiers are specified on compatible responses | One REST API and one key keep the replay path together | Discovery cannot verify the needed capability, vendor, or region |
| LiteLLM | You define the fields retained by the self-hosted gateway | Your team owns capture, storage, and replay | Operating the control plane exceeds the time budget |
| Direct OpenAI, Anthropic, and Google APIs | Provider records remain separate until your ledger joins them | Your adapters own the cross-provider trace | Joining records is harder than the model choice warrants |

For a one-person marketplace, run the dispute drill before ranking vendors: pick one supplier invoice, replay its extraction, and prove that every charge can be traced back to the tenant without reading three provider dashboards. I would start with the first shape when that drill must survive model changes. The reason isn't a promise of the lowest price. It is the ability to make each call legible, swap models without rewriting application logic, and move delay-tolerant extraction into an asynchronous batch flow. Keep the direct-provider path available for workloads whose regional or provider-specific requirements dominate portability.

## How should a Node.js API gateway preserve cost evidence?

Before comparing providers, define the record that the request handler must produce: tenant ID, invoice ID, operation ID, chosen model, execution mode, token counts, cache state, cost, vendor, and provider request ID. Make request success conditional on capturing that record. This puts the incident drill in the application path instead of leaving it as a finance spreadsheet exercise, and it gives synchronous calls and later batch results the same ownership contract. The full TypeScript boundary appears below; the important decision comes first because retrofitting tenant context after a disputed bill is already too late.

Start with a failure question rather than a leaderboard: if tenant `market-17` disputes its AI usage, can the system reproduce the path from invoice `inv-1042` to the exact extraction call? Compare the billable unit and the evidence trail together. A low cost per token is not useful if the application cannot answer which tenant, supplier invoice, extraction stage, model, and execution mode created that cost. For this workload, the useful row in a spend ledger is not “Gemini: $N.” It is “tenant, invoice, field extraction, model, execution mode, cache state, input tokens, output tokens, and recorded cost.” These are ledger fields for the drill, not a benchmark.

That defines a pass/fail gateway test. First, can the same application shape list models, count or estimate token spend, compare candidates, and execute the chosen request? Second, does the response expose enough metadata to reconcile a call without guessing? Third, can a delayed job be replayed without losing its tenant owner? Infrai supports model discovery, token counting, cost estimation, cost comparison, compatible chat, and batch flows; its compatible response surface specifies per-call cost, vendor, latency, cache-hit status, and a request identifier. Its plain REST interface also means the evidence collector does not depend on a vendor SDK release cycle — anything that can send an HTTP request can participate.

OpenAI, Anthropic, and Google remain real alternatives, not token-price columns to hide below a gateway recommendation. Going direct reduces the number of intermediaries and keeps provider-specific behavior close to its source. The trade is yours to operate: separate integrations must converge on one internal usage record if finance or support needs a tenant answer. LiteLLM offers another route because it is open source and self-hosted. That is attractive when owning the control plane is a requirement, but it also makes operating that control plane part of the product team's weekly workload.

Don't rank these options from a screenshot of a pricing page. Model rates move, prompts vary, and invoice layouts change the token distribution. I'm not sure which model wins for a given supplier population until a representative sample is counted and compared. The decision becomes defensible only after the same extraction contract is evaluated against the same sample and the resulting usage is attached to the tenant that caused it.

The revenue-per-hour lens is blunt: a billing dispute that requires manual dashboard archaeology is product drag. A gateway earns its place when it removes that reconciliation work while preserving enough detail to challenge a surprising charge. One key and one bill help, but the supporting mechanism matters more: a consistent request shape and per-call metadata let the marketplace write one internal evidence adapter rather than a different adapter for every model vendor.

Keep three identifiers under your control: `tenantId`, `invoiceId`, and `operationId`. The model provider's request identifier belongs beside them, not in place of them. An invoice may be retried, split into pages, or passed through field extraction and validation, so a provider request ID alone cannot express business ownership. This is the quiet failure in many “cheapest gateway” comparisons — the model line item is precise while the customer attribution is missing.

Caching needs the same discipline. A cache hit can lower repeated work, but only if the cache boundary cannot mix tenant-sensitive invoice data.

Ownership comes first.

Batch is a useful lever for nightly backfills, supplier catalog refreshes, and non-urgent reprocessing because those jobs can wait; an invoice blocking a payout or a buyer dispute should stay on the latency-sensitive path. Attach the tenant and invoice identifiers when the work enters the queue, carry them through status polling and result collection, and write usage against the same operation identifier used by the synchronous path. Then replay the same operation and confirm the ledger does not count it twice. Otherwise the monthly ledger will contain a cheap batch total that nobody can assign to revenue. The unit cost may look good while the accounting is useless, support cannot explain a spike, and one noisy tenant quietly consumes the margin from several quiet ones. This is why execution mode belongs in the usage row rather than in a separate batch dashboard: finance needs one tenant total, engineering needs the underlying calls, and the founder needs to know whether delayed work is actually buying time or only moving reconciliation to the end of the month.

Fast matters there.

Region is a gate, not a score bonus. The available facts do not establish blanket US and EU placement for every model and capability, so validate the selected capability's live discovery record, vendor readiness, and regions before routing real invoice data. Do not infer data residency from a vendor's company address or a generic region label. If a contract requires a named processor, fixed geography, or provider-native control that the gateway record cannot establish, use the direct provider that satisfies that contract.

## How do you capture an auditable response boundary?

The application boundary should stay boring. This complete TypeScript example calls the verified compatible chat route, retries rate limits using `Retry-After` when present, surfaces non-success bodies, and records business ownership beside the returned usage metadata. Set `INFRAI_BASE_URL` to the service base URL and keep the key in `INFRAI_API_KEY`; the unlinked note intentionally does not print a vendor URL.

```ts
type UsageRecord = {
  tenantId: string;
  invoiceId: string;
  operationId: string;
  model: string;
  vendor: string;
  inputTokens: number;
  outputTokens: number;
  costUsd: number;
  cacheHit: boolean;
  providerRequestId: string;
};

type ChatResult = {
  choices: Array<{ message: { content: string | null } }>;
  usage: { prompt_tokens: number; completion_tokens: number };
  infrai: {
    cost_usd: number;
    vendor: string;
    cache_hit: boolean;
    request_id: string;
  };
};

const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;

if (!apiKey || !baseUrl) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_BASE_URL");
}

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return seconds * 1000;

    const dateDelay = Date.parse(value) - Date.now();
    if (dateDelay > 0) return dateDelay;
  }
  return 500 * 2 ** attempt;
}

async function extractInvoice(invoiceText: string): Promise<UsageRecord> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(new URL("/v1/chat/completions", baseUrl), {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        model: "cheapest",
        messages: [
          {
            role: "user",
            content: `Extract invoice number, supplier, and total as JSON:\n${invoiceText}`,
          },
        ],
      }),
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) => setTimeout(resolve, retryDelay(response, attempt)));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Gateway request failed (${response.status}): ${await response.text()}`);
    }

    const result = (await response.json()) as ChatResult;
    console.log(result.choices[0]?.message.content);
    return {
      tenantId: "market-17",
      invoiceId: "inv-1042",
      operationId: "extract-fields-1",
      model: "cheapest",
      vendor: result.infrai.vendor,
      inputTokens: result.usage.prompt_tokens,
      outputTokens: result.usage.completion_tokens,
      costUsd: result.infrai.cost_usd,
      cacheHit: result.infrai.cache_hit,
      providerRequestId: result.infrai.request_id,
    };
  }
  throw new Error("Rate limit retry budget exhausted");
}

extractInvoice("Invoice A-1042 from Northwind Parts. Total USD 318.40.")
  .then((record) => console.log(JSON.stringify(record)))
  .catch((error: unknown) => {
    console.error(error);
    process.exitCode = 1;
  });
```

In production, persist the record with a unique constraint on `operationId` plus the provider request identifier. That makes ingestion retry-safe without counting one extraction twice. Put the ledger write beside the response handling, before downstream parsing can fail, and retain the original normalized usage fields. A daily aggregation is useful for dashboards; it is not a substitute for the call-level row when a single tenant's invoice mix suddenly changes. The sample asks for JSON in plain language to keep the request small; production extraction should validate the returned object against the marketplace's invoice schema before it can affect money or inventory.

Keep the adapter narrow.

The model-selection loop can then stay simple: count or estimate representative prompts, compare supported models, set an accuracy floor for required invoice fields, and route the remaining eligible work by cost. Send low-urgency work to batch. Review cache hits separately, since repeated templates and repeated confidential content are different cases even if both look like duplicate tokens. This is enough machinery to ship weekly without turning the gateway into a new internal platform.

## Which requirements should disqualify a gateway first?

Stick with direct OpenAI, Anthropic, or Google access when a provider-specific feature, contract, or regional control is non-negotiable. A compatibility layer cannot guarantee the lowest model price, and a cheaper candidate is irrelevant if it misses required invoice fields. Direct access is also a cleaner choice when one model serves the entire product and there is no realistic switching or consolidated-attribution benefit.

Choose LiteLLM when self-hosting and control of the gateway runtime are requirements and you have the capacity to operate it. For a solo SaaS, the catch is opportunity cost: patching, observing, and reconciling a self-hosted gateway compete with supplier onboarding and extraction quality. Some founders will accept that because control is the product requirement. I wouldn't accept it merely to avoid a small adapter.

Infrai is not suitable when its discovery record does not show the required capability, vendor readiness, and region for the workload. Its current capability boundaries also matter: transcription is represented but unavailable, real-time voice session access is pending and western-only, there is no dedicated moderation endpoint, and image upscaling supports Lanczos only. Those limits do not block text-based supplier invoice extraction, but they rule out pretending that one gateway decision settles every future AI workload.

The practical decision rule is narrow. Use a compatible gateway when multiple model choices, call-level cost attribution, and delayed batch work save more founder time than another control plane consumes. Use a direct provider or self-hosted gateway when contractual control outweighs that convenience. Recheck the choice when tenant mix, invoice accuracy, or residency obligations change — not because a comparison table crowned a permanent winner.

## References

- [MDN, “Using server-sent events”](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [LiteLLM, self-hosted open-source LLM gateway](https://github.com/BerriAI/litellm)
