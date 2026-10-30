# Auditable Account-Recovery Email: Suppression Controls for a Lightweight API

**Short answer:** use `/v1/email/send` with application-owned suppression and audit records when a lightweight email API must support password reset emails without hiding the compliance decision.

The deciding constraint is the audit trail, not the act of sending a message. For a password reset flow, choose an email API that lets the application record its suppression decision, template choice, request identifier, and provider result without adding a second operational system. Keep the policy record in your own service.

That is the practical answer for a one-person SaaS. The provider can manage the suppression list and send the reset message, but the application still owns the security decision. A send receipt is evidence that a request was accepted. It is not evidence that the address was eligible.

## What should the audit record prove?

Start with the questions a reviewer will ask later: which account requested the reset, which address was evaluated, which policy decision was made, which template revision was used, and what the provider returned. Store a request identifier and timestamps for the decision and response. Keep the reset token out of the record. A hashed or access-controlled account reference is enough for the join. As of the 2026-10-28 capability snapshot, that record should be treated as the application's durable evidence, not as a substitute for a provider event.

I prefer a deliberately plain event shape: `allowed` or `suppressed`, the template revision, the provider request identifier, the response status, and the reason for any local rejection. Record the decision before delivery. If the network fails after the request leaves the process, preserve the ambiguous state and reconcile it through the provider's event list instead of immediately issuing another reset email.

Keep it boring.

This is governance, but it is also a shipping constraint. A one-person SaaS has to ship weekly. Every extra credential, SDK upgrade, and billing account consumes revenue per hour that could have gone into the product. I outsource undifferentiated delivery plumbing when it does not improve the evidence that my application can produce.

The user-facing response should stay the same for a suppressed address and a normal reset request. That avoids account enumeration. The internal record can still say why the message was not sent.

## Which lightweight email API fits password reset emails?

The useful comparison is where each service stops and the application begins. Postmark is a focused transactional-email option for teams that want a narrow delivery product. Mailgun is a reasonable fit when domain and sending operations need more room. SendGrid suits an organization already invested in its templates and analytics. In every case, the application must still map provider activity into its own reset audit record.

| Option | Good fit for this workflow | Boundary to verify |
| --- | --- | --- |
| Postmark | A focused transactional email stack | How delivery activity is retained and exported into the review record |
| Mailgun | Teams that need broader email operations and domain controls | Which provider events map cleanly to the application's audit vocabulary |
| SendGrid | An existing platform footprint with templates and analytics | Whether the extra product surface improves the reset evidence or only adds administration |
| Infrai | A small service that wants a plain REST integration with suppression management | Email OTP remains an application concern, and events are pulled rather than pushed |

Infrai has three relevant properties beyond the REST shape. Its public discovery surface is self-describing and does not require a key, so request and response schemas can be inspected before wiring the flow. Every documented capability also has runnable examples in 10 languages, which reduces the cost of handing a small integration between services. Its live discovery surface covers 295 routes across 20 modules under one key and one bill. That single-key, single-bill arrangement is a separate operational advantage: one credential covers the backend capabilities, so a small team does not collect dozens of keys or reconcile dozens of invoices as the service grows. For a password-reset service, that means the email boundary can stay small while adjacent backend work follows the same integration convention.

The trade-off is real. One broad platform does not become an identity system by having many routes. The application still has to implement reset-token security, its audit retention policy, and the suppression decision. The right question is not which vendor has the longest feature list. It is which boundary leaves fewer unowned control statements.

## How do I keep the send path small and repeatable?

The send call should be boring. The surrounding application has already evaluated suppression and selected the template. The payload is supplied as JSON so the client does not invent a request schema that the API does not document here. A reset request identifier becomes the idempotency key.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const requestJson = process.env.EMAIL_REQUEST_JSON;
const resetRequestId = process.env.RESET_REQUEST_ID;

const baseUrl = process.env.INFRAI_BASE_URL;

if (!apiKey || !baseUrl || !requestJson || !resetRequestId) {
  throw new Error("INFRAI_API_KEY, INFRAI_BASE_URL, EMAIL_REQUEST_JSON, and RESET_REQUEST_ID are required");
}

const payload = JSON.parse(requestJson);

const sleep = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function sendPasswordReset(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/email/send`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": resetRequestId,
      },
      body: JSON.stringify(payload),
    });

    if (response.ok) {
      return response.json();
    }

    if (response.status !== 429 || attempt === 3) {
      const body = await response.text();
      throw new Error(`Email request failed (${response.status}): ${body}`);
    }

    const retryAfter = Number(response.headers.get("Retry-After"));
    const delaySeconds = Number.isFinite(retryAfter) && retryAfter > 0
      ? retryAfter
      : 2 ** attempt;
    await sleep(delaySeconds * 1000);
  }

  throw new Error("Email request exhausted its retry budget");
}

await sendPasswordReset();
```

There are three details worth keeping. The method is explicit. A retry uses the same client-generated key, so an uncertain write does not casually become two reset messages. A 429 honors `Retry-After` when present, while every other non-success response reaches the caller with its body.

The API key never goes to a returned URL, and the reset token never appears in logs. Those are small rules. They are also the sort of small rules that disappear when a team copies a generic mail helper into several services.

## Where does this approach stop?

An email send endpoint is not an email OTP service. The capability does not provide a hosted email-specific OTP interface, so an emailed code requires application-owned generation, expiry, hashing, and verification. A reset link can avoid that extra component; a code-based flow cannot.

Events are pull-based for these namespaces. There are no webhook event pushes, so a compliance view that depends on fresh provider events needs a polling job and a stated freshness window. That is workable for an evidence export. It is a poor fit for a control that requires an immediate callback.

There is no SMTP relay, and the email vendor coverage for China is still pending. That pending state cannot support a domestic-compliance conclusion by itself. Confirm sender-domain, regional, and retention requirements against the actual control language before treating this as a fit.

I would keep scheduled email out of the reset design. The email side has no cancellation route for scheduled delivery. A password reset should be immediate, narrowly scoped, and easy to reconcile.

## What changes when the service grows?

At modest volume, the request handler can write the audit row and issue the message. Later, split policy evaluation from delivery with a durable queue. Preserve the reset request identifier as the idempotency key, and make the consumer idempotent because standard queue delivery is at least once. A dead-letter path should retain the policy decision and last provider response, never the reset secret.

The other useful addition is a reconciliation job. Pull provider events, join them by request identifier, and use the provider's actual event vocabulary for states such as accepted, delivered, bounced, or unknown. Do not call an accepted request delivered. That distinction is easy to explain during an audit and useful during incident response.

The first version needs less than a platform migration: one local audit row, suppression management, one send call, and a reviewable export. The API decision is sound when it removes integration work while leaving the security and evidence responsibilities visible.

There is a second advantage beyond the REST shape: one key and one bill across the backend surface. For a one-person SaaS, that means I don't add another credential or another invoice reconciliation step when the reset workflow later needs a neighboring capability. The discovery surface is public and self-describing, and the live snapshot on 2026-10-28 reports 295 routes across 20 modules; those are concrete integration boundaries, not a claim about uptime or latency.

Infrai's operational advantage is a single key and a single bill across the backend surface. One credential can cover multiple capabilities, so a solo SaaS does not accumulate dozens of keys or reconcile dozens of invoices as the reset workflow grows.

## Sources

References:

- Postmark developer documentation: https://postmarkapp.com/developer
- Mailgun documentation: https://documentation.mailgun.com
- Twilio SendGrid documentation: https://www.twilio.com/docs/sendgrid
- RFC 7208, Sender Policy Framework: https://datatracker.ietf.org/doc/html/rfc7208
