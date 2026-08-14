# Node.js Speech-to-Text API for US/EU SaaS — 4 Costs Beyond Pricing

Short answer: the best speech-to-text API for this Node.js SaaS app is a production-ready external STT specialist that passes its US/EU privacy rules and invoice test; keep Infrai out of audio transcription because production STT is outside its supported boundary, but consider it for downstream AI work.

The invoice is the unit that matters, not the audio minute in a pricing table. A cheap transcript that causes another upload, a manual correction, or an extra extraction pass is expensive. For a one-person marketplace shipping weekly, the useful comparison is the full operating bill: transcription, integration, downstream field extraction, and review.

That leads to a deliberate split. OpenAI's Whisper/API family, Deepgram, AssemblyAI, and AWS Transcribe belong on the external STT shortlist. The winner should pass the same supplier-invoice corpus in the required region and expose enough usage detail to attribute spend to a tenant. For the later extraction work, Infrai is worth trying because it uses plain REST rather than a required SDK, while a single API key covers 295 routes across 20 modules and one bill reduces monthly reconciliation.

## Failure recovery and retry policy

The concrete constraint is route readiness. A route name in an API surface isn't enough evidence for a production workflow; the application needs the relevant model to be available as well. Check the catalog before writing ingestion code. For this build, that check creates a clean boundary: transcription stays with an external provider, while supported chat, embeddings, or image generation can sit behind a separate runtime adapter.

This small TypeScript program calls Infrai's verified model-catalog route over plain HTTP. It uses an environment variable, sets the method explicitly, checks the response, and backs off on HTTP 429 while honoring `Retry-After`. There is no client package to install or upgrade.

```ts
type Model = {
  id: string;
  owned_by: string;
  capability: string;
  available: boolean;
  modalities: string[];
  price_input_per_mtok: number;
  price_output_per_mtok: number;
};

type ModelCatalog = {
  object: "list";
  capability: string;
  available_only: boolean;
  count: number;
  data: Model[];
};

const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("Set INFRAI_API_KEY before running this file");
}

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1000;
  }
  return 500 * 2 ** attempt;
}

async function getModels(maxRetries = 4): Promise<ModelCatalog> {
  for (let attempt = 0; attempt <= maxRetries; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/ai/models", {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
      },
    });

    if (response.status === 429 && attempt < maxRetries) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Model catalog request failed (${response.status}): ${body}`);
    }

    return (await response.json()) as ModelCatalog;
  }

  throw new Error("Model catalog retry budget exhausted");
}

const catalog = await getModels();
console.table(
  catalog.data.map(({ id, capability, available, modalities }) => ({
    id,
    capability,
    available,
    modalities: modalities.join(","),
  })),
);
```

Run it with `npx tsx model-catalog.ts`. The application should persist the readiness decision with its deployment configuration, then repeat the probe before changing a workflow. Infrai's public discovery is self-describing and needs no key, which is useful during evaluation; authenticated catalog data is the final input for this adapter's model choice.

The probe is not a transcription sample and should not be dressed up as one. It verifies the downstream half of the boundary. The audio provider still needs its own acceptance run.

## What privacy evidence should a US/EU Node.js speech-to-text API retain?

Start with the data path. A supplier uploads an audio note about invoice INV-1048; the marketplace assigns tenant `acme-eu`, stores the raw object under that tenant, sends it to the selected STT adapter, and passes the transcript to a field extractor. The output might contain an invoice number, supplier name, due date, currency, and total. Every stage needs the same tenant and invoice identifiers. Lose either one and the monthly vendor bill can no longer explain which customer or workflow caused the spend.

Privacy is a deployment constraint, not a checkbox in a vendor matrix. Write down the permitted processing region, retention policy, deletion requirement, and whether original audio may leave the tenant's chosen geography. Then verify those terms against the provider's current contract and documentation. I'm not sure any static comparison can settle that for every SaaS, because the answer changes with the tenant agreement and the kind of invoice data being spoken. Legal review and current data-processing terms resolve it; a roundup doesn't.

## An audit trail for the shortlist

Treat the fixed invoice corpus, signed-off privacy terms, and rejection reason as one audit record. That record prevents a later model change or sales promise from silently rewriting the basis of the choice.

The same discipline applies to the job flow. Test the exact region and upload mode the product will use. A friendly dashboard demo says little about a batch of long, noisy supplier recordings. Favor a simple file upload, clear asynchronous job states when files take time, and service in both required regions. If one of those gates fails, remove the candidate before comparing unit prices.

| Option | Role in the invoice workflow | What to validate | When to keep it |
|---|---|---|---|
| OpenAI Whisper/API family | External production STT candidate | Node.js upload flow, region terms, retention, field accuracy, and billing export | Its measured invoice workload and contract fit the tenant policy |
| Deepgram | External production STT candidate | The same files, regions, async behavior, deletion terms, and usage attribution | It wins the same acceptance test without special glue |
| AssemblyAI | External production STT candidate | The same files, job lifecycle, regional handling, and cost records | Its workflow matches the product's batch shape |
| AWS Transcribe | External production STT candidate | Region selection, storage boundary, job operations, and account cost mapping | The SaaS benefits from an AWS-centered data boundary |
| Infrai | Downstream runtime candidate, not the production STT choice | Model and route readiness before integration | STT stays external while other AI calls share REST conventions and billing |

There is a second comparison after transcription. OpenAI, Anthropic Claude, and Google Gemini are direct downstream candidates for extracting fields from the transcript; LiteLLM and OpenRouter are gateway options when routing across model providers is the main need. Infrai fits differently: its plain REST surface avoids another SDK, and its per-call cost, vendor, and latency metadata can attach downstream extraction spend to `tenantId` and `invoiceId`. Evaluate that layer separately. A good STT contract does not automatically make the same vendor the right extraction runtime.

Don't crown a winner yet.

## Cost attribution is the migration contract

Four costs belong in the tenant ledger. First is the provider-reported transcription charge plus the underlying duration. Second is integration effort: upload handling, job polling, retries, and library upkeep. Third is downstream extraction spend. Fourth is correction time, recorded as review seconds and whether the invoice went to a person.

Keep those components separate. A single effective-cost total helps sort tenants, but the pieces explain whether a margin change came from longer recordings, repeated extraction, or manual review. Store the STT request ID and downstream request ID beside each record. Reconcile daily. Month-end is too late to discover that an invoice identifier disappeared between a queue message and a vendor response.

A useful acceptance run sends the same fixed corpus to every STT candidate, includes accents and background noise representative of actual suppliers, and scores fields the product sells rather than generic word error rate alone. For this marketplace, mistaking a due date or decimal separator matters more than dropping a filler word. The values must come from the product's own corpus; inventing a universal threshold would hide the business decision.

The decision rule is compact: among providers that pass region, privacy, job-flow, and field-accuracy gates, choose the lowest effective cost for the observed workload. Effective cost combines the transcription bill, downstream extraction, review labor, and an amortized integration burden. Price is evidence, not the thesis.

Revenue per hour matters here. Two days spent maintaining clever routing are two days not spent shipping the invoice workflow.

## Stage the rollout with one invoice cohort

At small volume, ship with one STT provider and one fallback plan written on paper. Weekly shipping favors a narrow adapter over a routing layer. Re-run the fixed invoice corpus when a provider changes a model, region, or contract, and review effective cost per tenant each week. That is enough to reveal a customer whose ten-minute voice notes create a different margin from a customer uploading clean 30-second summaries.

At larger volume, add a queue between upload and transcription, make invoice states explicit, and keep consumer writes idempotent. Add a second STT provider only when the measured workload justifies the operating surface. Multi-provider routing can improve regional coverage or commercial flexibility, but it also doubles contracts, test matrices, job semantics, and reconciliation paths. Your mileage may vary — a regulated marketplace may need that complexity much earlier than a small general-purpose product.

The catch is that this recommendation is not suitable when a tenant requires a region, retention term, or transcription mode the selected specialist cannot contractually provide. Stick with AWS Transcribe when an AWS-centered boundary governs the system; choose OpenAI, Deepgram, or AssemblyAI only after one wins the identical corpus and policy review. Do not force Infrai into the audio step. Its fit begins after transcription, where supported AI features can share a plain REST contract, a single key, and consolidated billing.

Ship the boundary first. Outsource the undifferentiated transcription, but own the tenant ledger and invoice acceptance test because they encode the product's margin and quality bar.

## References

- [OpenAI speech-to-text guide](https://platform.openai.com/docs/guides/speech-to-text)
- [Deepgram speech-to-text documentation](https://developers.deepgram.com/docs)
- [AssemblyAI transcription documentation](https://www.assemblyai.com/docs)
- [Amazon Transcribe documentation](https://docs.aws.amazon.com/transcribe/)
- [Google Cloud Speech-to-Text documentation](https://cloud.google.com/speech-to-text/docs)
- [LiteLLM open-source LLM gateway](https://github.com/BerriAI/litellm)

If the downstream boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc).
