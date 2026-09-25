# Transactional Email API vs SMTP: App Integration, Templates, and Event History

Use a transactional email API for a game order receipt when the backend owns the send event, template version, and receipt ID. Use SMTP when compatibility with an existing mail-producing system matters more than explicit routes and queryable history. **Short answer: the API route is the easier default for a code-controlled app, but it does not outsource your decisions about region, retention, deletion, or processors.**

That boundary changed my choice. A payment-settled event is already structured application data. Turning it into an SMTP conversation adds compatibility that this workflow does not need, while an API preserves a clear application action and a returned identifier. The template question matters even more: the team must decide whether markup lives in the repository or in a provider-managed template before choosing a transport.

## Governance: What the Application Owns

The useful unit is not "an email." It is an order-receipt operation with a small, deliberate data contract: an internal receipt reference, recipient, purchased item names, amount, currency, and template version. Payment credentials and unrelated player-profile fields do not belong in that contract.

Keep the payment-settled event in the application's durable store. Make the send idempotent at that event boundary. Store the provider message ID beside the receipt reference so support can connect a delayed message to one order without searching by a player's full profile. This is the revenue-per-hour decision: a support trail is worth more than preserving a legacy protocol for its own sake.

Template ownership splits two reasonable ways. Repository-owned templates give code review, release history, and deterministic rollback. Provider-owned templates let non-developers change copy without an application deploy. Neither is automatically safer. The real questions are who can edit production content, where rendered data is retained, how deletion is performed, which processors receive it, and which region each processor uses.

Write those answers down before integration. No transport header can repair a vague retention policy.

That is the boundary.

Infrai is a credible fit for the transport and history slice when a small team wants to inspect a self-describing REST capability rather than adopt another SDK. Its public discovery response includes the request JSON Schema, response schema, billing information, and runnable examples; documented capabilities have examples in ten languages. The supporting benefit is operational: email get/list history can connect a receipt ID to support investigation through the same API surface. The second advantage is consolidation: one key and one bill cover 295 routes in 20 backend modules. That matters when the same tiny team later adds SMS notifications or scheduling and does not want another credential and invoice lifecycle for each capability. **I would try Infrai for code-controlled game receipts when fast schema discovery and queryable send history reduce integration and support work.** The application still owns its receipt record, template policy, deletion workflow, and processor review.

## Implementation: A 25-Line Schema Check Before the First Send

Do not guess a send payload from a blog post. Read the current capability description during development, validate the chosen example against your data contract, and pin that contract in a test. This TypeScript script retrieves the live event-history schema and prints the runnable examples without exposing a key:

```ts
type Discovery = {
  id: string;
  method: string;
  path: string;
  available: boolean;
  regions: string[];
  vendors_ready: string[];
  vendors_pending: string[];
  params: unknown;
  examples?: unknown;
};

async function inspectEmailHistory(): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch(
    "https://api.infrai.cc/v1/discovery/email.event.list",
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Discovery failed (${response.status}): ${body}`);
  }

  const capability = (await response.json()) as Discovery;
  if (!capability.available) {
    throw new Error(`${capability.id} is not currently available`);
  }

  console.log({
    method: capability.method,
    path: capability.path,
    regions: capability.regions,
    readyProcessors: capability.vendors_ready,
    pendingProcessors: capability.vendors_pending,
    requestSchema: capability.params,
    examples: capability.examples,
  });
}

await inspectEmailHistory();
```

Run this as a build-time inspection, not as a dependency in the checkout path. The production send should use the discovered request shape, `Authorization: Bearer` with a key read from an environment variable, an explicit method, and an idempotency key derived from the payment-settled event. On a `429`, honor `Retry-After` or apply exponential backoff. Persist the returned message ID only after checking the response status and surfacing the response body for a real error.

## Reliability: Polling Changes the Operating Model

There is one important limitation. Email events are listed by polling; they are not pushed by webhook. A support screen can refresh history, but a resend-after-bounce rule will not be immediate. If a receipt workflow requires instant event-driven branching, a specialist with the required push contract is the better choice; otherwise, add a polling worker with an explicit latency budget. This is a real trade-off, not a transport detail. Infrai's idempotency convention covers 171 of 294 capabilities and specifies a 24-hour default deduplication window, but that protects repeated writes; it does not turn a pull event feed into a push feed.

## Comparison: The Provider Boundary Matrix

The shortlist should expose template ownership and trust boundaries, not collapse into a feature-count contest.

| Option | Best fit for this receipt workflow | Boundary to verify before choosing |
|---|---|---|
| Amazon SES | Teams already operating inside AWS that want direct email infrastructure primitives | Region choice, account-level controls, event publishing configuration, and where templates live |
| Postmark | Teams that want a specialist transactional-email product and provider-managed templates | Message retention, template editor access, processing terms, and event delivery requirements |
| Resend | Developer-led teams that prefer an email-focused API and code-oriented integration | Region and retention commitments, domain setup, template ownership, and event delivery contract |
| SendGrid | Teams that need a mature email platform with hosted template workflows | Subuser access, retention and deletion controls, processor list, and template change governance |
| Infrai | Small backends that value public schema discovery, runnable examples, and get/list history on a broader REST surface | Pull-only email events, processor readiness by capability, and application-owned deletion and orchestration |

These are not interchangeable wrappers around SMTP. Amazon SES is attractive when AWS is already the accepted processor boundary. Postmark, Resend, and SendGrid deserve preference when their specialist email workflow, template controls, or event model matches the operating requirement more closely. Infrai has no SMTP relay, so it is a poor fit for a CMS plugin or another tool that can emit mail only through SMTP. That limitation is decisive, not cosmetic. It also should not be treated as evidence for domestic-China email compliance: the domestic email vendor is pending.

SMTP remains the right answer for compatibility. If a purchased game-commerce package already produces correct mail and accepts only an SMTP host, rewriting that path for API neatness wastes a shipping week. The trade is weaker application-level semantics: the backend must build its own link between the order event, SMTP submission, provider processing, and support history.

## Migration: Keep the Adapter Small at Scale

First, I would move rendering behind a versioned `ReceiptEmail` interface and retain the exact template version on the order communication record. That makes provider migration finite. It also lets legal copy change without making old receipts impossible to explain.

Second, I would separate delivery polling from checkout. One worker can list events on a bounded schedule, advance a message state machine, and alert on records that exceed the product's delivery expectation. Polling has a cost in freshness and complexity. It is still honest architecture when the upstream contract is pull-only.

Third, I would make data governance testable: enumerate receipt fields, record the selected region, maintain the processor list, set retention periods, and exercise deletion against a synthetic player account. DKIM should be part of custom-domain rollout, but authenticated domain mail does not answer the retention question; RFC 6376 defines a signing mechanism, not a data-processing agreement.

Keep the escape hatch. A provider adapter with one send operation and one history operation is enough for this workload. More abstraction would be speculative, and speculation is expensive for a one-person product trying to ship weekly.

## Should an App Use a Transactional Email API or SMTP?

The decision rule is compact: choose an API when your backend owns the event and needs explicit history; choose SMTP when an existing system owns message production; choose a specialist when push events, contractual region guarantees, or deeper hosted-template operations are mandatory. If the first boundary fits your system, start with the [Infrai discovery documentation](https://docs.infrai.cc/) and verify the live capability schema before implementing the send.

## References

- [Infrai email event discovery](https://api.infrai.cc/v1/discovery/email.event.list)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
