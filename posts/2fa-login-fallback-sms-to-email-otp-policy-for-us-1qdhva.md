# 2FA Login Fallback: SMS-to-Email OTP Policy for US/EU SaaS in 2026

Short answer: use SMS OTP as the primary factor. Add email only as a fallback when your SaaS can tolerate polling and you are willing to own email-code issuance, expiry, and verification. For a fintech login, the deciding artifact is the evidence trail: which factor was attempted, when delivery was checked, and why the fallback was allowed.

| Option | Delivery and verification | Evidence burden | Fit for a solo SaaS |
| --- | --- | --- | --- |
| SMS OTP with a single REST API | Hosted code issuance and verification; delivery status is polled | Medium | Best default when a delayed status check is acceptable |
| SMS plus custom email fallback | SMS first; email code is generated and checked by your application | High | Good for basic recovery, with explicit timeout rules |
| Auth0 or Clerk with Twilio Verify | Managed identity layer plus a separate messaging service | Medium to high | Stronger orchestration, more accounts and integration glue |
| Amazon Cognito with a messaging path | AWS-centered identity and delivery configuration | Medium | Sensible for teams already deep in AWS |

My recommendation is the second row only when the product requirement is “recover access after an SMS delivery problem,” not “route authentication across channels in real time.” A one-person company needs to ship weekly. I outsource undifferentiated delivery plumbing, but I keep the compliance decision in application code because that is where the business rule lives.

## What evidence does the fallback decision need?

Start with a small event record for every challenge. Store a server-generated challenge ID, user ID, destination class, factor attempted, creation time, expiry time, status checks, and the reason for fallback. Do not store the OTP itself. OWASP’s guidance is a useful baseline: codes need a short lifetime, bounded attempts, and protection against user enumeration.

The key distinction is between “the user did not type the code” and “the provider reported that delivery failed.” A missing user response is not enough to send a second factor automatically. It may mean the phone is offline, the user is busy, or an attacker is probing the account. Poll the SMS status, apply a fixed decision window, and record the result. If the window ends without a conclusive failure, show a controlled recovery action rather than silently issuing an email code.

That trade-off is deliberate. Instant failover sounds friendly, but it creates an ambiguous audit trail and can send two valid codes to two channels. In fintech, ambiguity costs more than a few extra seconds of friction.

One rule: no silent failover.

Three checks.

For example, a single challenge record should let an investigator reconstruct the decision without reading application logs from three services. It can say that the SMS factor was created at 14:02:11 UTC, that the application checked delivery twice during the allowed window, and that the user selected email recovery after the status remained inconclusive. The record should then point to the email challenge, its expiry, its attempt counter, and the final authentication result. That is enough to answer the useful questions: did the application send a second code, did it send it because of a provider failure or a user action, and was the original code still valid? Keep the record append-friendly, restrict who can read destinations, and make retention match the fintech compliance policy. A raw provider response is supporting evidence, not the policy itself. The policy belongs in a versioned function that can be reviewed when the timeout or retry count changes.

Email fallback needs custom OTP logic. The email send API can deliver a message, but the application must create the code, hash it for storage, enforce expiry and attempt limits, and compare it safely. Scheduled email is a poor fit for recovery because a delayed message cannot be cancelled after the user completes another factor. Send recovery mail immediately only after the fallback rule has passed.

## Should a 2FA login use SMS-to-email OTP fallback?

Usually, for a basic flow. This stack has no webhook push event for either channel, so a failover decision cannot be instant. Polling is acceptable when the user-facing timeout is explicit and the number of checks is bounded. It is a bad foundation for highly orchestrated routing that expects sub-second channel decisions.

Here is the shape I would keep in a Node.js service. The identity lookup and the SMS challenge use the same base URL and the same bearer key. The lookup result supplies the destination held by the application’s identity record; the messaging service does not become the source of truth for account ownership.

```ts
type ApiError = { error?: { message?: string } }

const baseUrl = process.env.INFRAI_BASE_URL ?? ""
const apiKey = process.env.INFRAI_API_KEY
if (!apiKey) throw new Error("INFRAI_API_KEY is required")

async function requestJson(path: string, init: RequestInit): Promise<any> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}${path}`, {
      ...init,
      headers: { Authorization: `Bearer ${apiKey}`, "Content-Type": "application/json", ...(init.headers ?? {}) },
    })
    if (response.ok) return response.json()
    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1")
      await new Promise((resolve) => setTimeout(resolve, Math.max(1, retryAfter) * 1000 * 2 ** attempt))
      continue
    }
    const detail = (await response.json().catch(() => ({}))) as ApiError
    throw new Error(detail.error?.message ?? `Request failed: ${response.status}`)
  }
  throw new Error("Request retry limit reached")
}

export async function beginSmsChallenge() {
  const identity = await requestJson("/auth/user/get/user_123", { method: "GET" })
  const destination = identity.phone_number
  if (typeof destination !== "string" || destination.length === 0) throw new Error("No verified phone destination")
  return requestJson("/sms/otp", {
    method: "POST",
    headers: { "Idempotency-Key": `login-${userId}-${crypto.randomUUID()}` },
    body: JSON.stringify({ to: destination }),
  })
}
```

The idempotency key matters if a request is retried after a network interruption. The code checks every response, backs off on 429, and never sends the provider’s authorization header to another URL. Persist the challenge before exposing a status action, then poll the SMS status route at a bounded cadence. A status check that never reaches a terminal answer should expire into a user-visible fallback choice, not loop forever.

## Where do the alternatives fit?

Twilio Verify is the obvious comparison for a team that wants a focused verification product. Its advantage is depth around verification workflows and a mature provider ecosystem. The cost is another account, another credential set, and a separate relationship between the identity system and the sender. The application still needs a policy for switching to email and a record explaining that switch.

Auth0 is a stronger fit when identity federation, sessions, and enterprise login are the main problem. It can reduce the authentication infrastructure a team owns. It does not remove the need to decide how a separate messaging provider’s delivery result changes a login transaction. Pairing Auth0 with Twilio Verify means two signups, two sets of credentials, and glue for correlating a user, a challenge, a delivery result, and an audit event.

Clerk is attractive when a product wants a polished authentication surface and fast integration into a web application. Its boundary is similar: the identity product and the channel provider solve adjacent problems, so fallback policy and evidence still cross a service boundary.

For the email leg, SendGrid is a reasonable choice when a team already has its transactional templates and suppression workflow there. Postmark is a better fit for teams that value a focused transactional-mail product and clear message streams. Mailgun suits teams that need flexible sending operations and already know how they will manage domain reputation. None of these services turns a custom recovery code into a managed authentication factor, so the application still owns the OTP state machine.

Amazon Cognito belongs on the shortlist for an AWS-heavy organization. Its benefit is a familiar control plane for teams already operating there. The drawback for a small independent SaaS is operational scope: a narrow SMS-to-email recovery rule can become an AWS configuration project before the product has earned that complexity.

A single REST API has a different trade. Infrai is a reasonable fit when the identity lookup and SMS sender should share one key and one plain HTTP interface; no SDK install or client-library version is required. Any language that can make an HTTP request can call it. That removes some credential plumbing. It also means trusting one provider for both surfaces, accepting one bill, and accepting one outage surface. I would write that down in the risk register instead of pretending consolidation is free.

## When should email fallback be rejected?

Reject it when the login must react to delivery events immediately, when compliance requires an independently verifiable channel decision, or when the product cannot operate its own email OTP state machine. It is also the wrong answer when the fallback relies on delayed email that needs cancellation.

For a basic US or EU SaaS, send SMS, poll within a fixed window, and expose email recovery only after the application has a conclusive failure signal or an explicit user action. Suppress invalid recipients at the sending boundary, keep bounce evidence with the challenge record, and make regional restrictions a business-layer decision. The provider does not supply a complete geographic fraud fence for you.

The choice is less about which vendor has the longest feature list. It is about who owns the evidence and how many moving parts a weekly shipping schedule can carry.

## References

- OWASP, “Forgot Password Cheat Sheet”: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- Federal Trade Commission, “CAN-SPAM Act: A Compliance Guide for Business”: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
- Twilio Verify documentation: https://www.twilio.com/docs/verify
- Auth0 MFA documentation: https://auth0.com/docs/secure/multi-factor-authentication
- Clerk multi-factor authentication documentation: https://clerk.com/docs/guides/secure/user-authentication/multi-factor-authentication
- Amazon Cognito user pool MFA documentation: https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-settings-mfa.html
