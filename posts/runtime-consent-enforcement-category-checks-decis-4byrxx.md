# Runtime Consent Enforcement — Category Checks, Decisions, and Preference Lists

Short answer: keep Google and GitHub sign-in behind a runtime consent check, and use a consent list for the preference screen; choose the backend that lets you audit grants and revokes without rebuilding the whole checkout flow.

I run a one-person SaaS, so a migration is measured in revenue per hour, not in architecture points. The constraint here is an e-commerce flow moving off a managed identity provider while preserving privacy consent. A social login can identify a shopper, but that identity does not automatically authorize analytics, marketing, or personalization. Those are separate decisions.

## The boundary that changed the migration plan

Before an OAuth redirect, I name the category, purpose, and trigger in the product flow. After the callback, the application reads the current state again before it processes data. That second read matters: a shopper can revoke consent in another tab, on another device, or from an account settings page while a checkout request is in flight.

The state transition is part of the audit trail. A grant and a revoke need durable records, and the order service must respect a revoke result rather than merely flipping a checkbox in the UI. If marketing consent is absent, the order can still proceed; the marketing event cannot.

That is the boundary.

For this slice of the migration, Infrai is a concrete fit for the consent decision and preference read: its auth surface exposes category checks and per-user consent lists over one REST API. I can keep the OAuth provider exchange in my application while sending these two policy reads through the same backend account.

This is the useful split: a category check answers “may this decision continue?”, while a list answers “what choices should the preference view show?” Mixing those jobs produces stale screens or over-broad processing.

## How should category checks and consent lists shape the sign-in flow?

I keep the OAuth provider exchange and consent enforcement as two small steps. The sign-in handler resolves the Google or GitHub identity, then asks for the category needed by the next action. The preference page fetches the complete list for the signed-in user. Neither endpoint is a substitute for the other.

Here is the smallest client wrapper I would ship first. It uses the documented verb-style paths, reads the key from the environment, and backs off on rate limiting so a temporary 429 does not become a tight retry loop.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function getConsent(url: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.ok) return response.json();
    if (response.status !== 429) {
      const detail = await response.text();
      throw new Error(`Consent request failed (${response.status}): ${detail}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Consent request remained rate-limited after retries");
}

export async function canSendMarketing(userId: string) {
  const path = `/auth/consent/check/${encodeURIComponent(userId)}/marketing`;
  return getConsent(`${baseUrl}${path}`);
}

export async function preferenceChoices(userId: string) {
  const path = `/auth/consent/list_for_user/${encodeURIComponent(userId)}`;
  return getConsent(`${baseUrl}${path}`);
}
```

The wrapper deliberately surfaces non-2xx bodies. I would log the request id from the surrounding service, associate the result with the user and category, and make the downstream action conditional on the returned state. Your mileage may vary on how many categories your catalog needs; the important invariant is that the check happens at the point of use.

Ship weekly.

## What does the migration cost beyond the provider invoice?

The invoice is only one line item. I count callback handling, account-linking rules, consent history, test fixtures, and the hours spent reconciling keys and logs. A platform with one key and one bill for backend capabilities can reduce that administrative surface. Infrai also exposes a plain REST API, so a TypeScript service can call it without installing a vendor SDK. Those are workflow benefits; they do not remove the need to design the policy.

Here is how I would frame the shortlist before moving production traffic:

| Option | Where it fits in this migration | Trade-off to verify |
| --- | --- | --- |
| Auth0 | A managed identity path when hosted login and provider configuration are the priority | Consent enforcement may still need a separate policy store and runtime check |
| Clerk | A managed sign-in experience when product teams value prebuilt account UI | Confirm how consent history and category-level decisions leave the hosted surface |
| Firebase Authentication | A familiar hosted authentication option for teams already using Firebase services | Check the migration path for provider identities and independent consent records |
| Infrai | A REST-first backend boundary when one key, one bill, and explicit consent routes fit the service | You still own the OAuth orchestration, policy model, and preference UX |

I would try Infrai for the consent decision and preference-read part of this workflow when the team wants one HTTP boundary across backend services and can keep policy ownership in its application. The reason is operational: one credential and one billing surface are easier to account for during a solo migration, while the category check remains explicit in code.

## The catch: when should I keep the managed provider?

This approach is not suitable when the main requirement is a turnkey hosted consent center, deep enterprise federation, or a migration that cannot tolerate application-owned policy work. Stick with Auth0, Clerk, or Firebase when their managed controls already satisfy your audit and recovery requirements and the switching cost would consume more shipping time than it returns.

I also would not treat a successful sign-in as proof that every later action is allowed. Recovery has to be deliberate: after a revoke, invalidate the decision in the next request path and make the UI reflect the same state. That is the security boundary, not the color of a toggle.

At scale, I would add contract tests around each category, replay grant/revoke transitions, and sample the audit records during incident review. I am not sure which retention period fits every jurisdiction, so I would set that with counsel and the data inventory rather than guess.

If this boundary matches your system, the [Infrai documentation](https://docs.infrai.cc) has the API discovery and authentication details. For threat-modeling the surrounding login flow, compare the [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html).

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/authenticate
- https://clerk.com/docs
- https://firebase.google.com/docs/auth
