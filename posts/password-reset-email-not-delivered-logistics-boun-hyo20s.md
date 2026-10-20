# Password Reset Email Not Delivered: Logistics Bounce and Spam Folder Triage

TL;DR: For a logistics SaaS, treat a missing password reset as a delivery-state problem before touching the template or changing senders. Record the provider acceptance ID, classify the eventual outcome, suppress invalid recipients after a permanent failure, and inspect SPF, DKIM, and DMARC alignment from the received message. This is the least complex route to reliable recovery mail.

| Evidence | What it tells you | Next move |
|---|---|---|
| No acceptance ID | The application-to-mail handoff failed | Retry safely and inspect the enqueue boundary |
| Permanent bounce | The recipient or destination rejected the message | Suppress the address and ask for a corrected one |
| Accepted, then filtered | Handoff worked; mailbox policy or reputation intervened | Inspect authentication results and message content |
| Delivered event, no inbox mail | The mailbox accepted it, but placement is uncertain | Check spam, tabs, rules, and the exact destination address |

**Recommendation:** choose a bounce-led runbook over a provider swap. It preserves the evidence needed to separate an application fault from a recipient fault, and it prevents repeated sends to an address already known to be invalid. For a solo operator shipping weekly, that boundary matters more than a longer feature checklist.

## Why was the password reset email delivered to the spam folder?

An API success means one narrow thing: the sending service accepted the request. It does not prove inbox placement. A later permanent bounce, a temporary deferral, mailbox filtering, or a user rule can produce the symptom "I never got the reset email." Collapsing those outcomes into one `sent` boolean destroys the useful part of the trail. The tempting assumption is that acceptance and delivery are the same event. They are not. For one reset request, correlate application enqueue, transport acceptance, bounce or delivery, and token consumption. If the chain ends at acceptance, the application worked but the investigation is unfinished. If it ends before acceptance, mailbox policy is a distraction. This distinction keeps a support ticket from turning into a blind DNS change.

Keep 4 timestamps: requested, accepted, final delivery outcome, and consumed. Also keep 1 provider message identifier and a normalized recipient hash for correlation. Do not log the reset token or full reset URL. A recovery link is a credential.

Keep the evidence.

For logistics software, the operational cost is sharper than it looks. A dispatcher locked out during a pickup exception cannot wait while support repeatedly presses resend. Yet repeated sends to a permanently invalid address are the wrong response. **Reliability starts with stopping futile traffic.**

DMARC adds a separate question: does an authenticated identifier align with the domain visible to the user? RFC 7489 defines alignment for SPF-authenticated and DKIM-authenticated identifiers and describes policy discovery through DNS. A message may show an SPF or DKIM pass while still failing DMARC alignment. Read the complete authentication result; do not stop at the first green word.

## The two criteria that earn their keep

The first criterion is outcome fidelity. Can the system distinguish temporary from permanent failure, deduplicate repeated events, and retain the raw reason long enough to investigate? A hard bounce should move an address into suppression. A transient failure should remain retryable within a bounded policy. These states must not share a catch-all `failed` bucket. There is a real trade-off: append-only delivery evidence uses more storage and needs retention controls, while a current-state row is cheaper to query but loses the path that support needs. For a reset flow, retain the small event trail and derive current state from it.

The second is identity alignment. The visible From domain, the DKIM signing domain, and the SPF-authenticated path have different jobs. DMARC passes when its published rules are satisfied through an aligned authenticated identifier. That makes alignment a better diagnostic question than "Did we turn DKIM on?"

Small systems benefit from boring choices. One event table, one suppression table, and one idempotent consumer are enough. Outsource transport if it is undifferentiated, but own the state transition because support, security, and login availability all depend on it. That is a good revenue-per-hour trade.

Not every system needs this design. Its limitation is the extra event storage and consumer logic. A low-volume internal tool with a human administrator may be better served by direct log inspection, while a product without reliable delivery callbacks must begin with mailbox headers and transport logs. Bounce-led triage is also unsuitable when the visible symptom spans every valid recipient immediately after a domain change; authentication-first analysis is faster there.

## Model the bounce boundary once

The event consumer below deliberately uses a generic interface. It accepts out-of-order duplicates, records evidence before acting, and blocks a known-invalid recipient. The exact event names vary by transport, so normalize them at the adapter edge.

```ts
type DeliveryEvent = {
  eventId: string;
  messageId: string;
  recipientHash: string;
  kind: "accepted" | "delivered" | "temporary_bounce" | "permanent_bounce";
  occurredAt: string;
  reason?: string;
};

interface DeliveryStore {
  hasEvent(eventId: string): Promise<boolean>;
  append(event: DeliveryEvent): Promise<void>;
  suppress(recipientHash: string, reason: string): Promise<void>;
}

export async function applyDeliveryEvent(
  store: DeliveryStore,
  event: DeliveryEvent,
): Promise<void> {
  if (await store.hasEvent(event.eventId)) return;

  await store.append(event);

  if (event.kind === "permanent_bounce") {
    await store.suppress(
      event.recipientHash,
      event.reason ?? "Permanent delivery failure",
    );
  }
}
```

The subtle failure is ordering. A delivered event can arrive after a temporary bounce, and the same callback can arrive twice. Append-only evidence plus idempotency handles both without inventing a fragile sequence. Permanent suppression should be reversible through a deliberate address-correction flow, not an automatic timer.

Duplicates happen.

Test the boundary with fixtures for duplicates, reordered events, missing optional reasons, and a permanent bounce after acceptance. Then deploy authentication changes gradually. DMARC policy and aggregate reporting exist at the domain level, so a rushed DNS edit can affect more mail than the reset path being debugged.

## When is the runner-up approach better?

Header-first inspection is the runner-up. Use it when the recipient can produce the actual message from spam or quarantine. Its Authentication-Results fields can expose alignment and evaluation details that an application log cannot see. This path is especially useful when events say delivered but placement is poor across one mailbox domain.

Start with authentication-first analysis, rather than bounce state, when a new sending domain is being introduced or when every valid recipient at several destinations begins filtering mail at once. The blast radius points toward identity configuration, not individual addresses.

A provider migration belongs later. Consider it only after the application handoff, bounce classification, suppression behavior, domain alignment, and mailbox evidence are understood. Otherwise the migration erases the baseline and changes several variables at once. Shipping a new transport adapter may feel productive; it rarely answers why the original message vanished.

**A compact operating rule:** trace one reset request from request ID to acceptance ID to final outcome. If it bounced permanently, suppress and correct the recipient. If it was accepted without a final outcome, preserve the evidence and investigate the transport. If it was accepted by the mailbox, inspect placement and authentication alignment.

Do not send another reset until you know which branch you are in. That single constraint keeps an urgent support case from becoming a deliverability problem. It also fits a one-person release cadence: the state machine stays stable while adapters, DNS settings, and mailbox behavior change around it.

## Further reading

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- US A2P 10DLC compliance documentation, useful when a recovery flow adds an SMS fallback: https://www.twilio.com/docs/messaging/compliance/a2p-10dlc
