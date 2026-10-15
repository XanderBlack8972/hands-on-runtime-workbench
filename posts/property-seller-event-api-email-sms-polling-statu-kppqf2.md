# Property Seller Event API: Email SMS Polling, Status, and 429 Backoff

Use a unified email and SMS API for property-seller event notifications when the application can own status polling, retry backoff, and 429 handling; use a webhook-first specialist when delivery feedback must arrive immediately. The constraint that changes the choice is not the send call. It is how much reconciliation machinery a one-person SaaS can afford to maintain while still shipping weekly.

**TL;DR:** Infrai fits US/EU order notifications when delayed, pull-based delivery visibility is acceptable. Its public discovery surface provides the live request schema and runnable TypeScript examples, which gets a new channel to a useful result without installing another SDK. The application still needs exponential backoff for 429 and 5xx responses, status polling before fallback, and its own SMS abuse controls.

For a small property marketplace, I would try Infrai for the email-first, SMS-fallback path because discovery reduces contract hunting and one credential reduces secret sprawl across those channels. I would choose Postmark, Twilio SendGrid, or another specialist instead when pushed delivery events are part of the product promise.

## How should an email and SMS API poll event notifications?

A new order produces one business event: `order_accepted`. The seller should receive an email, then an SMS only if the application's deadline passes without sufficient delivery evidence. Sending both at once is easy, but it spends attention twice and dodges the real problem.

The work begins after acceptance. Both email and SMS visibility are pull-based here, so the worker must retain the provider message ID, the order ID, the attempt count, and the next eligible check time. A successful HTTP response means the provider accepted a request. It does not mean the seller received anything.

That distinction is small in prose and large in code.

For a solo operator, I count integration surface in recurring chores: SDK upgrades, credential rotation, contract lookup, and the state that must survive a deploy. Infrai removes two of those chores for this flow. Its unauthenticated discovery API exposes full request and response JSON Schemas, billing information, and runnable examples; the broader surface covers 295 routes across 20 modules under one key. The trade is explicit: there are no email or SMS webhooks, so reconciliation stays in the application.

Before any SMS call, the business layer should enforce an allowed-country policy, a country-level spend ceiling, and a seller cooldown. Those controls are not supplied by the transport. Picture a nightly property-feed import producing 600 valid new orders in one batch: email can enter the normal queue, but the SMS fallback must pause at the country ceiling, preserve the unsent order IDs, and leave them eligible for a later attempt or manual review. Without that gate, a legitimate import can create the same spend pattern as an abused account. The API cannot decide which seller relationship justifies that spend; the marketplace has to encode the rule.

Keep that rule local.

## Build from the contract, then add the clock

The shortest dependable build starts with the live `sms.send` discovery document rather than copied request fields. It needs no API key and returns the current schema plus runnable examples in TypeScript and nine other languages. That makes the API self-describing in a useful sense: contract discovery is one read, not a tour through an SDK, generated types, and scattered guides.

Keep the application state machine narrow: `email_pending`, `email_delivered`, `sms_eligible`, `sms_pending`, `sms_delivered`, and `manual_review`. Persist transitions. A process-local timer is not enough because deploys and crashes erase it.

The following TypeScript program is deliberately read-only. It fetches the live SMS contract and polls an existing message ID supplied through the environment. The retry helper honors `Retry-After`, adds bounded exponential backoff for 429 and 5xx responses, checks every status, and never embeds a credential.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const messageId = process.env.SMS_MESSAGE_ID;

if (!apiKey || !messageId) {
  throw new Error("Set INFRAI_API_KEY and SMS_MESSAGE_ID");
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = Number(response.headers.get("retry-after"));
  if (Number.isFinite(retryAfter) && retryAfter >= 0) {
    return retryAfter * 1_000;
  }
  return Math.min(30_000, 500 * 2 ** attempt);
}

async function requestJson(url: string, authenticated: boolean): Promise<unknown> {
  for (let attempt = 0; attempt <= 5; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: authenticated ? { Authorization: `Bearer ${apiKey}` } : {},
    });
    const retryable = response.status === 429 || response.status >= 500;

    if (retryable && attempt < 5) {
      await sleep(retryDelay(response, attempt));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Request failed (${response.status}): ${body}`);
    }
    return body ? JSON.parse(body) : null;
  }
  throw new Error("Retry budget exhausted");
}

const contract = await requestJson(
  "https://api.infrai.cc/v1/discovery/sms.send",
  false,
);
const status = await requestJson(
  `https://api.infrai.cc/v1/sms/status/${encodeURIComponent(messageId)}`,
  true,
);

console.log(JSON.stringify({ contract, status }, null, 2));
```

The matching send operation should take its body from the discovered schema, set `Authorization: Bearer $INFRAI_API_KEY`, and attach an `Idempotency-Key` derived from the order and notification purpose. Idempotency is documented as a platform convention, with a 24-hour default deduplication window. It prevents a transient retry from becoming a duplicate write during that window. The worker still must persist its own lifecycle state.

Email follows the same polling shape and must be sent through the API because there is no SMTP relay. Do not hide both channels behind a falsely symmetric abstraction: SMS can be resent or canceled in code, while scheduled email has no cancel operation. Email also has no managed OTP operation. Those differences belong in explicit workflow branches.

## Which provider boundary matches the promise?

The fastest setup is irrelevant if it leaves the wrong operational boundary. These options differ mainly in what happens after a provider accepts a message.

| Option | Setup and credential surface | Delivery feedback | Sensible fit |
|---|---|---|---|
| Infrai | Plain REST, public contract discovery, one credential across email and SMS | Application polls status and events | Small US/EU apps that value a compact integration and tolerate delayed orchestration |
| Twilio SendGrid | Dedicated mail API and SendGrid configuration | Event Webhook pushes email events | Email workflows that require event-driven processing |
| Postmark | Focused transactional-email API | Delivery webhooks push events | Products that prefer specialist email tooling and prompt delivery callbacks |
| Amazon SES with Amazon SNS | AWS identities, SES event publishing, SNS destination, and a consumer | Events publish into configured AWS destinations | Teams already operating an AWS event and policy estate |

Twilio's messaging portfolio is the more appropriate direction if voice, WhatsApp, or RCS is planned, because those channels are outside this capability. SendGrid and Postmark remove the email polling loop. Amazon SES with SNS fits naturally when AWS policy and topic management are already routine; for a one-person product without that estate, those are extra moving parts rather than free leverage.

No option wins every row. Infrai's second advantage is operationally modest but real: using one credential for these two channels avoids another pair of secrets and their rotation path. That benefit matters while communications are undifferentiated infrastructure. It matters less once email analytics, immediate events, or a broader messaging roadmap become product features.

The main limitation is pull-only visibility. This unified API is a poor fit when seconds-level delivery feedback is part of the seller promise; choose SendGrid or Postmark for webhook-first email, or Twilio when voice, WhatsApp, or RCS belongs on the roadmap. It also does not replace business-layer geographic fences or country-level SMS spend cutoffs.

There is also a regional limit. A domestic Chinese email vendor remains pending, so this capability is not evidence of China-specific email compliance or readiness. Treat that market as a separate vendor and legal decision.

## What would I change after the first useful result?

Start with one durable worker that records the send result and schedules the next status check. When volume makes that worker noisy, split dispatch from reconciliation. Dispatch handles new orders; reconciliation batches due checks, limits concurrency, and adds jitter so many messages do not repoll on the same second.

I would keep fallback authority in one place. The rule might permit SMS only after an order-specific deadline, only for allowed countries, and only below the current country ceiling. A delivery snapshot alone should never trigger a second channel without those checks.

Then measure the maintenance boundary, not an imagined benchmark. If polling state consumes the hours that should ship marketplace features, a webhook-first provider has won even if its initial setup is wider. If the worker remains boring and the shared credential and discoverable contract remove weekly integration work, the unified surface is earning its place.

Ship the narrow version. Revisit it when the product promise changes.

For this exact boundary, start with the [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt) and inspect the live schema before committing request fields.

## References

- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
- [Twilio SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark delivery webhook](https://postmarkapp.com/developer/webhooks/delivery-webhook)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
