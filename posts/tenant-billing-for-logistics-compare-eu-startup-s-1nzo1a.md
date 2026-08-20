# Tenant Billing for Logistics — Compare EU Startup Speech-to-Text API Pricing per Minute

Short answer: choose the speech-to-text API whose quote survives a tenant-level replay of real logistics moderation audio, then route every transcript charge to a tenant before classifying the report. A low advertised per-minute figure is not a decision if rounding, retries, storage, or missing tenant metadata cannot be reconciled.

| Choice | Use it when | Reject it when |
|---|---|---|
| Primary API behind a thin adapter | It wins the replay and produces reconcilable usage records | Its invoice units cannot be mapped back to a tenant |
| Runner-up behind the same adapter | It passes the same quality gate and provides a credible fallback | Switching requires changes to moderation business logic |
| Self-hosted transcription | Audio volume is steady and someone can own capacity, models, and EU operations | Operations would displace weekly product work |

The recommendation is deliberately conditional. For a one-person SaaS, the least complex acceptable option is a hosted API behind a small boundary, with an internal usage ledger as the source of tenant allocation. Outsource the undifferentiated transcription work; keep the cost evidence and moderation policy in your own system.

## Evaluate billing evidence before comparing quotes

Compare the bill you can reproduce, not the price label. Ask OpenAI, Deepgram, AssemblyAI, and Google Cloud for current written terms covering the same region, model class, processing mode, billing increment, minimum charge, channel treatment, retry treatment, taxes, and currency. Those products are candidates, not a ranking: this note does not have enough attributable pricing evidence to declare one cheapest, and current quotes are what would resolve that uncertainty.

Normalize each quote into the same workload. The input should be a frozen sample of voice reports from the logistics moderation queue, stripped of personal data or processed under the controls your legal review requires. Preserve duration, channel count, codec, tenant, and a content category label. Human reviewers should grade the resulting transcript only on errors that change the downstream moderation class: a misspelled depot name may be harmless, while dropping “not” is not.

The important denominator is accepted moderation reports, not uploaded minutes. Use one equation for every candidate:

`effective cost per accepted report = total attributable evaluation charge / reports that pass the transcription quality gate`

Suppose a tenant submits one 61-second report. If a quote rounds units, the evaluation ledger must use the billed quantity defined in that quote rather than silently treating the report as 1.017 minutes. This is a test fixture, not a claim about any named provider. The same rule applies to stereo files, empty audio, duplicate callbacks, and client retries. Tiny mismatches compound when a tenant opens hundreds of cases, and “roughly right” is useless when support asks why its allocation moved.

Don't mix transcript token counts with audio billing units. A tokenizer such as `tiktoken` applies byte-pair encoding to text; it can help estimate a later language-model step, but it does not establish how an audio API bills transcription. Keep those two ledger lines separate.

## What should EU startups require from tenant-level speech-to-text API pricing?

Cost visibility starts before upload. Generate an internal report ID, require a tenant ID, record the audio duration measured by your ingest path, and create one immutable attempt record for each outbound transcription request. When a provider returns usage data, store the raw reported unit and your normalized unit alongside it. Never overwrite the original observation.

This boundary pays for itself when classification gets more complicated. The moderation pipeline can retry a transient client-side network failure, reject unsupported input, or send a low-confidence transcript for human review without losing the chain from invoice to tenant. It also prevents the classifier from becoming the place where billing rules, provider response shapes, and logistics policy collide — three concerns that change for different reasons.

Use explicit states: `received`, `transcribing`, `ready_for_review`, `classified`, and `rejected`. An attempt also needs an idempotency key. If the same report enters twice, the worker should locate the existing attempt instead of charging for duplicate work. Fail closed when `tenantId` is absent with an internal code such as `TENANT_CONTEXT_MISSING`; a transcript without ownership is an accounting defect even if its words are perfect.

Short rule: no tenant, no call.

## Developer experience belongs at one typed boundary

The adapter should expose the information the application owns and hide the response shape it does not. Keep it boring. This example avoids any commercial endpoint or SDK, and the values returned by a concrete adapter must come from that provider's documented response and your current contract.

```ts
type TranscriptRequest = {
  tenantId: string;
  reportId: string;
  attemptId: string;
  audio: Uint8Array;
  measuredSeconds: number;
};

type TranscriptResult = {
  text: string;
  providerRequestId: string;
  reportedQuantity: number;
  reportedUnit: "second" | "minute";
};

interface SpeechAdapter {
  transcribe(input: TranscriptRequest): Promise<TranscriptResult>;
}

type UsageEntry = {
  tenantId: string;
  reportId: string;
  attemptId: string;
  providerRequestId: string;
  measuredSeconds: number;
  reportedQuantity: number;
  reportedUnit: TranscriptResult["reportedUnit"];
  recordedAt: string;
};

async function transcribeForReview(
  adapter: SpeechAdapter,
  request: TranscriptRequest,
  appendUsage: (entry: UsageEntry) => Promise<void>,
): Promise<string> {
  if (!request.tenantId) throw new Error("TENANT_CONTEXT_MISSING");
  if (request.measuredSeconds <= 0) throw new Error("INVALID_AUDIO_DURATION");

  const result = await adapter.transcribe(request);

  await appendUsage({
    tenantId: request.tenantId,
    reportId: request.reportId,
    attemptId: request.attemptId,
    providerRequestId: result.providerRequestId,
    measuredSeconds: request.measuredSeconds,
    reportedQuantity: result.reportedQuantity,
    reportedUnit: result.reportedUnit,
    recordedAt: new Date().toISOString(),
  });

  return result.text;
}
```

The write order is intentional: don't release text to the classifier until the usage entry is durable. In a production queue, make the transcript and ledger transition atomic through a database transaction or an outbox pattern. A worker crash after the external call but before persistence is the awkward case; reconcile it by provider request ID and idempotency key according to the selected provider's documented guarantees. I'm not sure which reconciliation field will be present in a future contract, so I would make that field a procurement gate rather than guess.

The classifier itself should consume plain transcript text plus domain context. It should not know which transcription service ran. That keeps a weekly shipping cadence realistic: changing a speech candidate means implementing one adapter and replaying the corpus, not reopening the moderation workflow.

## Reliability means reconciling unknown outcomes

Start the evaluation with a shadow run. Send the same approved corpus to each candidate under comparable settings, but do not let any result affect a live moderation decision. Record wall-clock completion, quality-gate outcome, reported usage, locally measured duration, retry count, and tenant attribution. Then reconcile the evaluation export against the provider's billing artifact.

The order matters. A candidate that produces excellent text but cannot support tenant reconciliation creates manual work every month. That work has a revenue-per-hour cost, and it arrives on the same calendar as feature delivery. I would reject an unreconcilable candidate before spending time tuning its classifier prompt.

Use a few hostile fixtures, too:

- a 0-byte object rejected before the external call;
- a 61-second mono report and a 61-second stereo report;
- two deliveries with the same idempotency key;
- a network timeout where the final remote disposition is initially unknown;
- an accented depot name and a sentence whose meaning flips if “not” disappears.

The timeout fixture is where many clean diagrams stop being useful. The client cannot assume that no response means no processing. Mark the attempt `reconciliation_pending`, do not fire an unbounded retry, and resolve it using documented request lookup or billing evidence. This describes a general distributed-systems failure mode, not a service outage. Your mileage may vary because contracts expose different evidence; the pass condition is that the team can reach one auditable outcome without reading application logs by hand.

## Governance needs an exit trigger before procurement

Stick with the runner-up when its transcript passes the same moderation gate and its usage evidence makes tenant allocation materially easier to operate. The nominal per-minute leader is not suitable when billing increments distort your short-report workload, EU processing terms fail legal review, invoice exports cannot be tied to request IDs, or the supported audio path forces pre-processing that your tiny team must own.

Self-hosting is the other legitimate runner-up. It fits steady, high utilization and teams prepared to own model serving, capacity planning, security updates, observability, and regional data handling. The catch is the operator. If that operator is also the person shipping paid features every week, variable API spend may be the cleaner business choice even when a spreadsheet makes compute look attractive.

Do not promise permanent portability. Audio formats, timestamps, diarization, confidence values, and usage metadata differ across contracts. A narrow adapter limits the blast radius; it doesn't erase semantic differences. Keep the replay corpus, scoring rubric, and ledger schema stable so a future decision is an evidence update instead of a rewrite.

## Further reading

- https://github.com/openai/tiktoken
- https://python.langchain.com/docs/integrations/chat/openai/
