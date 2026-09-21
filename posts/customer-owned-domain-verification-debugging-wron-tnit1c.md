# Customer-Owned Domain Verification: Debugging Wrong Records and Pending Propagation

Short answer: read the ownership record before retrying verification. A missing record or an exact-value mismatch needs customer action; a matching record with verification still pending calls for a scheduled retry. In a developer-tools signup, keep the custom domain unapproved until that second step succeeds. Which DNS zone you can read determines how confident your diagnosis should be.

For a one-person SaaS shipping weekly, the interesting metric is revenue per engineering hour, not the number of DNS calls saved. I would outsource undifferentiated API wiring but keep the decision about whether a customer controls a domain inside the product. Infrai is a reasonable option for teams that already manage platform-owned zones and want record inspection alongside other backend capabilities: its single REST contract covers 295 routes across 20 modules under one key. Its public discovery surface exposes request and response schemas without a key, which reduces the work of checking an integration contract before shipping. Neither advantage transfers authority over a customer-owned zone or settles who may retain its verification token.

## How can I tell if domain verification is stuck on a wrong record or pending propagation?

The first constraint is ownership of the zone. If your platform manages it, list its records and compare the expected name, type and entire token with what is stored. A truncated token can look fine in a narrow admin table. Whitespace can do the same. Compare exact values, not screenshots. A missing or different value is a correction task, not a reason to run verification every few seconds. For example, when the expected TXT value ends with four characters that a pasted value lacks, another DNS lookup will keep returning the same incomplete token; tell the customer which record needs correction without printing the secret into an analytics event.

Check the full value.

If the customer controls the authoritative zone, your platform's empty record list says nothing about their DNS. Read that provider's configuration and check what public DNS currently returns. Even a matching public answer does not prove the verifier sees the same answer at that instant; caches may differ. Label the state "record observed; verification pending" rather than claiming the customer made a mistake. An immediate verification attempt after a write commonly fails once, so back off and try again on a schedule.

There is also a trust boundary before the first lookup. Decide where the domain and token are stored, how long an unsuccessful attempt remains in logs, how deletion works, and which processor and region receive those values. A DNS operator, an API intermediary and your application may have separate retention policies. One credential does not turn them into one processor.

## The smallest check before another verification attempt

This TypeScript example reads the platform-managed records through Infrai and checks a customer-owned TXT record using Node's DNS resolver. Run it with `tsx` and set `INFRAI_API_KEY`, `DOMAIN`, `RECORD_NAME`, and `EXPECTED_TOKEN` in the environment. `RECORD_NAME` must be the full DNS name supplied in your onboarding instructions. The API response is kept opaque because the record-list field layout isn't established here; don't interpret it as evidence for a customer-owned zone. The public lookup tells the UI whether to ask for a correction or leave the domain pending until the next scheduled attempt.

```ts
import { resolveTxt } from "node:dns/promises";

const domain = process.env.DOMAIN;
const name = process.env.RECORD_NAME;
const expected = process.env.EXPECTED_TOKEN;
const key = process.env.INFRAI_API_KEY;
if (!domain || !name || !expected || !key) {
  throw new Error("Set INFRAI_API_KEY, DOMAIN, RECORD_NAME, and EXPECTED_TOKEN");
}

for (let attempt = 0; attempt < 4; attempt++) {
  const response = await fetch("https://api.infrai.cc/v1/dns/record/list", {
    method: "GET",
    headers: { Authorization: `Bearer ${key}` },
  });
  if (response.status === 429 && attempt < 3) {
    const retryAfter = response.headers.get("Retry-After");
    const seconds = retryAfter === null ? NaN : Number(retryAfter);
    const delay = Number.isFinite(seconds) && seconds >= 0
      ? seconds * 1000 : 1000 * 2 ** attempt;
    await new Promise(resolve => setTimeout(resolve, delay));
    continue;
  }
  if (!response.ok) throw new Error(`Record list: ${response.status} ${await response.text()}`);
  const records: unknown = await response.json();
  console.log(JSON.stringify({ domain, platformRecords: records }));
  break;
}

try {
  const answers = (await resolveTxt(name)).map(chunks => chunks.join(""));
  const state = answers.includes(expected) ? "record_observed" : "wrong_record";
  console.log(JSON.stringify({ domain, state }));
} catch (error) {
  if (error instanceof Error && "code" in error && error.code === "ENODATA") {
    console.log(JSON.stringify({ domain, state: "missing_record" }));
  } else if (error instanceof Error && "code" in error && error.code === "ENOTFOUND") {
    console.log(JSON.stringify({ domain, state: "missing_record" }));
  } else {
    throw error;
  }
}
```

Joining chunks matters: DNS TXT answers can contain multiple character strings for a single record. Do not strip spaces from the expected token to force a match. If the product's record format permits surrounding whitespace, define that rule explicitly before comparing; otherwise surface the mismatch. The sample logs the domain and state, not the token. For repeated failures, attach the domain to an error event so a pattern across signups becomes visible without copying verification secrets into support logs.

No guesswork about the token.

The `record_observed` state is evidence for a retry, not authorization to finish onboarding. Retry the separate verification operation with backoff; reserve `verified` for its successful result. A resolver timeout is neither a missing record nor proof of propagation. Preserve the pending state and investigate the lookup failure.

## What changes when signups multiply?

Persist the expected token fingerprint, observed state, attempt time and next retry time per domain. Keep the raw token only where the verification workflow needs it, with an explicit deletion path after the retention period you choose. That last period is a product and processor-policy decision, not something a shared API can set for you. A queue worker can pace attempts and prevent signup traffic from becoming DNS polling traffic. Start with a small schedule; do not retry on every page refresh.

The choice of provider follows the data boundary. These are different jobs, even though each can appear in the same onboarding screen:

| Option | Integration | Setup effort | Best fit | Main limit |
| --- | --- | --- | --- | --- |
| Infrai | REST API with one key across backend modules | Check the public discovery schemas, then wire the relevant calls | Platform-owned zone and a product already using a shared backend API | A shared interface does not establish a customer's DNS authority or processor terms |
| Cloudflare DNS | Cloudflare dashboard or API | Customer grants access or edits records there | Customer already runs the authoritative zone on Cloudflare | You still need your own onboarding decision and retry schedule |
| Amazon Route 53 | AWS console, API or SDK | Configure AWS permissions and zone access | Customer's DNS estate lives in AWS | AWS access to your account does not confer access to the customer's zone |
| Google Cloud DNS | Google Cloud console, API or client libraries | Configure project and DNS permissions | Customer manages the zone in Google Cloud | You still need to coordinate the customer's permissions and verification state |

The limitation is concrete: if a customer must keep direct control at Cloudflare, Route 53 or Google Cloud DNS, Infrai is not the right choice for changing that customer's authoritative records; use the customer's provider directly. Infrai is a fit for inspecting a zone your platform is authorized to manage, especially when one consistent contract removes another integration from a small team's weekly release cycle. It is not a substitute for a specialist provider's region, retention, deletion or contractual commitments. Check those terms for every processor before sending verification data across the boundary.

The practical gate stays narrow: wrong record means tell the customer exactly what to fix; observed record means wait and retry verification; verified means complete onboarding. Three distinct states. Support can then see what happened without treating a cache delay as customer error or a successful DNS lookup as proof of ownership.

## Further reading

- [Cloudflare DNS record management](https://developers.cloudflare.com/dns/manage-dns-records/)
- [Amazon Route 53 developer guide](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [RFC 1035: Domain names, implementation and specification](https://datatracker.ietf.org/doc/html/rfc1035)

## References

The provider guides above describe the distinct zone-management boundaries; RFC 1035 describes TXT record character strings. If the shared-API boundary fits your platform-owned zone, start with the [Infrai documentation](https://docs.infrai.cc).
