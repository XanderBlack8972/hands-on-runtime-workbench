# Password Recovery: Brand-Safe Accessibility, Dark Mode, and Node.js Rendering

Short answer: build a password reset email template from one typed content model, render accessible HTML and plain text from it, and block release when the two versions disagree.

| Choice | Accessibility risk | Brand control | Maintenance cost | Best fit |
|---|---:|---:|---:|---|
| One content model, two renderers | Low | High | Medium | Most small SaaS teams |
| HTML composed separately from text | Medium | High | High | Regulated review with separate owners |
| Text-only message | Low | Limited | Low | Internal or deliberately minimal systems |

The first choice is the default. The important boundary is not HTML versus text. It is whether the security meaning, expiration copy, and recovery URL can drift between formats. A one-person SaaS gets better revenue per engineering hour by making that drift mechanically difficult, then spending human review time on the copy a customer will actually read.

## What makes a recovery message safe to act on?

A password reset email has one job: let the intended recipient recognize a requested action and reach a controlled recovery flow. It should not make the recipient decode marketing language, inspect a decorative image, or trust urgency alone. Put the product name, the action, the expiration statement, and the "ignore this" path in visible text. Don't ask for a password, one-time code, or reply by email.

Keep the subject literal: "Reset your Acme account password" is easier to evaluate than "Urgent security alert." The body can be equally plain. "We received a request to reset the password for your Acme account" states what happened without claiming who initiated it. The next sentence should say how long the link remains usable, but the duration must come from the same server policy that creates the token. Hard-coding "15 minutes" in a template while the token service uses another value creates a security and support problem.

Brand safety is mostly restraint. Use the legal product name consistently, send from a domain the company controls, and keep promotional modules out of account recovery mail. Google documents authentication and delivery expectations in its email sender guidelines; those controls belong in the sending system, not as badges or reassuring claims inside the message. A polished logo cannot compensate for an unauthenticated sender.

Keep it boring.

Recovery also needs a lifecycle outside the template. Generate a random, single-purpose token; store or otherwise protect it according to the authentication design; enforce expiration on the server; invalidate it after successful use; and avoid logging the full URL. NIST SP 800-63B is useful framing for authenticator lifecycle and recovery risk, but the assurance target and threat model still determine the exact flow. I'm not sure a template review alone can prove that flow sound. A token-state integration test and a security review can.

This is the first decision criterion: the email must reflect server truth. If a preview has current colors but stale expiry copy, it fails.

## Which HTML, text, accessibility, and dark mode rules matter most?

Start with a reading order that works without CSS. Use a real heading, ordinary paragraphs, and a descriptive link label such as "Reset your password." Do not make "click here" the only clue. Keep the recovery URL visible in the plain-text alternative so a user can inspect or copy it, and never rely on color alone to distinguish the action from surrounding copy.

Dark mode is an enhancement, not a second design system. Declare support with the `color-scheme` metadata and provide explicit foreground, background, link, and button colors under `prefers-color-scheme: dark`. Then inspect the result with images disabled and styles removed. Email clients don't interpret every style identically, so your mileage may vary; that is exactly why the unstyled reading order and text version carry the core meaning.

Contrast deserves a real preview, not a hex-code glance. A white wordmark with a transparent background may disappear on a white canvas, while a dark logo tile can turn into a black rectangle after client-side color treatment. Use a logo asset that remains identifiable in both schemes, include useful alternative text, and let text carry the product identity if the image is unavailable. No drama. The action still works.

The second decision criterion is parity under degradation. Review these states before release: normal HTML, simulated dark mode, images off, narrow viewport, and plain text. This is a short checklist because it should run every week, not become a quarterly design project. Outsource the undifferentiated rendering work to small deterministic functions; keep the security policy and copy review in the application boundary.

## How should a Node.js API preview password reset email HTML and text?

Use a typed input that contains copy facts rather than prebuilt fragments. The preview path should receive a fake token and a non-production origin, render both variants, and reject missing values. Production can call the same pure renderer after its own authorization and token creation steps. That separation keeps preview code from becoming a token issuer.

The focused TypeScript example below escapes all dynamic HTML, encodes the token as a query parameter, and derives the visible expiry line once. The text renderer consumes that same line. It deliberately does not choose a token lifetime or claim that a request succeeded; those are application decisions.

```ts
type ResetEmailInput = {
  productName: string;
  resetOrigin: string;
  token: string;
  expiresInMinutes: number;
};

type RenderedEmail = {
  subject: string;
  html: string;
  text: string;
};

const escapeHtml = (value: string): string =>
  value.replace(/[&<>"']/g, (character) => ({
    "&": "&amp;",
    "<": "&lt;",
    ">": "&gt;",
    '"': "&quot;",
    "'": "&#39;",
  })[character] as string);

export function renderPasswordResetEmail(input: ResetEmailInput): RenderedEmail {
  if (!input.productName.trim() || !input.token || input.expiresInMinutes <= 0) {
    throw new Error("E_RESET_EMAIL_INPUT");
  }

  const resetUrl = new URL("/account/recover", input.resetOrigin);
  resetUrl.searchParams.set("token", input.token);

  const productName = input.productName.trim();
  const expiry = `This link expires in ${input.expiresInMinutes} minutes.`;
  const subject = `Reset your ${productName} account password`;

  const html = `<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <meta name="color-scheme" content="light dark">
  <style>
    :root { color-scheme: light dark; }
    body { margin: 0; background: #ffffff; color: #202124; font: 16px/1.5 Arial, sans-serif; }
    main { max-width: 600px; margin: 0 auto; padding: 32px 20px; }
    a.action { display: inline-block; padding: 12px 18px; background: #0b57d0; color: #ffffff; }
    @media (prefers-color-scheme: dark) {
      body { background: #202124; color: #f1f3f4; }
      a { color: #8ab4f8; }
      a.action { background: #8ab4f8; color: #171717; }
    }
  </style>
</head>
<body>
  <main>
    <h1>Reset your password</h1>
    <p>We received a request to reset the password for your ${escapeHtml(productName)} account.</p>
    <p><a class="action" href="${escapeHtml(resetUrl.href)}">Reset your password</a></p>
    <p>${escapeHtml(expiry)}</p>
    <p>If you did not request this, you can ignore this email.</p>
  </main>
</body>
</html>`;

  const text = [
    "Reset your password",
    "",
    `We received a request to reset the password for your ${productName} account.`,
    `Reset your password: ${resetUrl.href}`,
    expiry,
    "",
    "If you did not request this, you can ignore this email.",
  ].join("\n");

  return { subject, html, text };
}
```

A preview handler can return these strings to an authenticated internal UI, but it should redact the token before logging. Use a fixed fixture such as `preview-token-not-valid` and an origin reserved for documentation. I don't ship when the preview check reports `E_RESET_COPY_PARITY`; the check means the required action, expiration line, or ignore copy appeared in only one representation. The useful response is to compare the structured render input first, then inspect the HTML heading, action label, expiration sentence, and ignore sentence beside their text equivalents. If the input is identical but one line is absent, fix the renderer and add an assertion for that field. If the input differs, fix the preview boundary so it invokes both renderers once with the same immutable object. Re-run semantic checks before opening a visual preview, because a screenshot can make two messages look similar while hiding a missing text line or a changed destination. Finally, inspect the parsed recovery URL host and confirm the fixture token never entered logs or artifacts. That error name is local policy, not an SMTP status; it does not describe mail delivery.

Test content invariants instead of taking screenshots alone. Assert that both variants contain the product name and expiration statement, that the HTML has one recovery link, and that its host matches the configured origin. Parse the markup in the test environment rather than searching it with a regular expression. Screenshot comparisons can catch visual regressions after those semantic checks pass, but a pixel match cannot tell whether the destination host is correct.

Keep the preview payload synthetic. Real reset links in issue trackers, snapshots, or CI artifacts turn a harmless design review into a credential-recovery exposure. Short logs win here.

Stop there.

## When is the runner-up approach the better choice?

Separate HTML and text composition is the runner-up, and the catch is obvious: it creates more review surface. It is still the better choice when policy requires independently approved channels, when the plain-text message must follow materially different legal copy, or when separate teams own each format. In those cases, keep a shared schema for security facts and add a parity test for action, product, expiry, and ignore instructions. Do not share rendered prose merely to claim there is one template.

Stick with text-only mail when the audience or operating environment intentionally rejects HTML, or when the team cannot maintain client previews. It gives up visual hierarchy and dark-mode branding control, but it retains the essential recovery path with much less rendering risk. That is a valid trade, especially for internal systems.

The single-model approach is not suitable when localization changes sentence structure so much that interpolated shared lines become unnatural. Share structured facts, not English fragments: product name, action URL, and expiration duration can remain common while each locale owns complete sentences. A native-language review matters more than deduplicating five lines of copy.

There is also a hard boundary around delivery. Template tests do not establish sender authentication, reputation, throttling, or inbox placement. Treat those as deployment checks, monitor delivery and complaint signals, and keep recovery endpoints rate-limited according to the application's threat model. Ship weekly, but don't confuse a green rendering snapshot with a working account-recovery system.

## References

- https://support.google.com/a/answer/81126
- https://pages.nist.gov/800-63-3/sp800-63b.html
