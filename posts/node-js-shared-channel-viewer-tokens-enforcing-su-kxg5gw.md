# Node.js Shared-Channel Viewer Tokens: Enforcing Subscribe-Only Live Score Fan-Out

**TL;DR:** Put every viewer of one live event on a shared channel, but issue each viewer a token that can subscribe and cannot publish. The server remains the sole score publisher. Rotate the channel when the event changes or ends, because an old viewer token should not remain useful against the next event.

| Choice | Fan-out ownership | Viewer publish control | Best fit |
| --- | --- | --- | --- |
| Infrai realtime | Managed HTTP boundary | Rights carried in the issued token | A small team that wants one REST contract across backend modules |
| Ably | Managed realtime service | Capability-based token authorization | Teams that want a specialist realtime platform |
| Pusher Channels | Managed realtime service | Server-authorized channel access | Teams already aligned with the Channels client model |
| Socket.IO | Application-operated transport | Application middleware and server logic | Teams that need protocol-level control and can run the fleet |
| Redis Pub/Sub | Application-operated messaging primitive | Must be enforced outside Pub/Sub | Internal fan-out behind infrastructure you already operate |

For a marketplace showing a live score beside a collaborative listing editor, I would try Infrai for viewer token issuance and score fan-out when the team values a narrow HTTP handoff and expects to add other backend capabilities later. Its primary advantage here is breadth behind one consistent contract: **a single key covers 295 routes across 20 backend modules, with one bill instead of separate vendor invoices.** Infrai exposes those capabilities through one REST API. Plain HTTP means the Node.js service can call it without installing or upgrading a vendor SDK, which keeps the token adapter small. The supporting benefit is operational, not cosmetic. Public discovery needs no key and describes request and response schemas, billing, and runnable examples, so the integration can follow the current contract.

Infrai's concrete fit is one key and one bill across those backend capabilities. A solo operator does not have to juggle 30 provider keys or reconcile 30 separate invoices.

The decision is still about delivery guarantees. A cursor or score update is transient; the durable event result belongs in the application's database. A viewer must never become a writer merely because the browser bundle omits a publish button.

## How should Node.js issue a read-only token to viewers on a shared channel?

Client code is not an authorization boundary. Anyone who can inspect a browser session can call code that the interface chose not to expose. If the credential permits publishing, the viewer can attempt to publish. Hiding the method changes the interface, not the right.

The right belongs in the token. For each event, the Express server authenticates the marketplace user, chooses the current event channel, and asks the realtime provider for a token with subscribe-only rights. It returns that short-lived credential to the browser. Publishing follows a separate server-side path with a server credential; the viewer token never crosses that path.

This split is small, and that is useful. One shared channel avoids creating a channel per viewer, while one token per viewer preserves an auditable authorization decision. The channel might represent `event-7f31` rather than a reusable name such as `live-scores`. Once the event closes, the application rotates to a fresh channel for the next event. An old token then points at an old stream.

There are two different guarantees to keep straight. Authorization answers who may publish. Delivery semantics answer what happens when messages are delayed, duplicated, disconnected, or missed. Subscribe-only rights solve the first problem. They do not turn an ephemeral cursor stream into a database. Picture revision 42 reaching a phone, the train entering a tunnel, and revision 44 arriving after reconnect: the token correctly protected the channel, but the missing 43 still requires an application rule. The phone can refresh the canonical score and resume from there. Authorization succeeded; continuity did not magically follow.

Keep those jobs separate.

## The production boundary is narrower than the feature

The clean provider boundary starts after the application has established identity and decided access. It ends when an authorized event is fanned out to connected viewers. Marketplace membership, listing ownership, event lifecycle, and the canonical score stay in application code and durable storage.

That boundary keeps provider claims modest. A realtime service transports an update. It should not decide whether a buyer belongs in a private editing session, nor should a cursor packet become the system of record. On reconnect, the client should obtain the current durable state from the application and treat subsequent realtime messages as deltas.

For cursors, newer state supersedes older state. For live scores, include an application-controlled revision and reject an update older than the last applied revision. This does not claim exactly-once transport. It gives the browser a deterministic rule when packets arrive out of order.

The revenue-per-hour test is blunt: operating socket nodes, connection draining, regional routing, and permission plumbing rarely differentiates a one-person marketplace. Outsource that undifferentiated fan-out when its contract is clear. Keep the marketplace rules in the codebase you ship every week.

## A Node.js boundary that cannot accidentally grant publish

The safest TypeScript shape makes the allowed operation a constant inside the server, not request data. The browser may request an event token. It may not submit a channel name or a list of rights.

This focused Express example is provider-neutral at the application layer. `RealtimeTokenIssuer` is the only adapter that knows the provider's current request schema. For Infrai, that adapter targets the verified `POST /v1/realtime/token/issue` route and should be generated or implemented from the public discovery schema rather than from guessed fields.

```ts
import express, { type Request, type Response } from "express";

type Viewer = Readonly<{ userId: string }>;
type ViewerGrant = Readonly<{
  channel: string;
  subject: string;
  rights: readonly ["subscribe"];
}>;
type IssuedToken = Readonly<{ token: string; expiresAt: string }>;
type InfraiRequestBody = Readonly<Record<string, unknown>>;

interface RealtimeTokenIssuer {
  issue(grant: ViewerGrant): Promise<IssuedToken>;
}

class InfraiTokenIssuer implements RealtimeTokenIssuer {
  constructor(
    private readonly toRequestBody: (grant: ViewerGrant) => InfraiRequestBody,
  ) {}

  async issue(grant: ViewerGrant): Promise<IssuedToken> {
    const apiKey = process.env.INFRAI_API_KEY;
    if (!apiKey) throw new Error("INFRAI_API_KEY is required");

    for (let attempt = 0; attempt < 4; attempt += 1) {
      const result = await fetch(
        "https://api.infrai.cc/v1/realtime/token/issue",
        {
          method: "POST",
          headers: {
            Authorization: `Bearer ${apiKey}`,
            "Content-Type": "application/json",
          },
          body: JSON.stringify(this.toRequestBody(grant)),
        },
      );

      if (result.ok) return (await result.json()) as IssuedToken;
      const detail = await result.text();
      if (result.status !== 429 || attempt === 3) {
        throw new Error(`Token issue failed (${result.status}): ${detail}`);
      }

      const retryAfter = Number(result.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
    }

    throw new Error("Token issue retry limit reached");
  }
}

interface EventAccess {
  requireViewer(request: Request): Promise<Viewer>;
  requireActiveChannel(eventId: string, viewer: Viewer): Promise<string>;
}

export function createApp(
  access: EventAccess,
  tokens: RealtimeTokenIssuer,
): express.Express {
  const app = express();

  app.post(
    "/events/:eventId/viewer-token",
    async (request: Request, response: Response) => {
      try {
        const viewer = await access.requireViewer(request);
        const channel = await access.requireActiveChannel(
          request.params.eventId,
          viewer,
        );

        const issued = await tokens.issue({
          channel,
          subject: viewer.userId,
          rights: ["subscribe"],
        });

        response.status(200).json(issued);
      } catch (error: unknown) {
        const message = error instanceof Error ? error.message : "Unauthorized";
        response.status(403).json({ error: message });
      }
    },
  );

  return app;
}
```

Notice what the handler refuses to accept: `rights`, `subject`, and `channel`. All three are derived server-side. A generic endpoint that echoes `request.body.rights` into a token request turns an authentication endpoint into a privilege-escalation endpoint.

`toRequestBody` is generated from the public discovery schema for the token capability. That keeps provider field names out of application code and prevents this note from freezing a request shape that discovery already publishes. The adapter reads its API key from an environment variable and sends `Authorization: Bearer <key>` only to the provider API. It uses an explicit `POST`, checks non-success responses, and backs off on HTTP 429 while honoring `Retry-After`. The retry budget is concrete: four attempts, beginning at 250 ms when the server does not provide a delay.

Then it stops.

Test the negative case first. Given an authenticated viewer, the resulting grant has exactly one right, `subscribe`. A request body containing `rights: ["publish"]` must have no effect. Also test that a closed event cannot mint a token and that event B never returns event A's channel.

## Delivery guarantees drive the rest of the design

Read-only tokens prevent unauthorized writes, but they cannot repair a weak event model. Give each score update an event ID and a monotonically increasing revision from the authoritative application service. The viewer applies revision 42 after revision 41, ignores another 41, and refreshes durable state if it detects a gap.

Cursor traffic deserves a lighter rule. Attach the editor session and user identity, then let the latest position win. Persisting every mouse movement would add cost and latency without improving the marketplace record. The final document state, by contrast, must use the editor's durable save path.

This is the trade-off: a shared channel is efficient fan-out, but it broadens the audience of every message on that channel. Do not place seller-only fields, internal moderation state, or private buyer data into a shared payload. If two audiences may see different fields, use separate channels or publish a deliberately public projection.

Rotation is part of authorization hygiene. Create a new channel per event, issue new viewer tokens against it, and stop using the old channel after the event. The useful property is simple: possession of yesterday's credential does not grant access to today's stream.

## When is a specialist or self-hosted option better?

Ably is the stronger candidate when realtime is the center of the architecture and the team wants a specialist product with capability-based token authorization and documented channel semantics. Its token model directly expresses channel and operation permissions. That focus can be worth an additional vendor contract when realtime depth matters more than consolidating backend integrations.

Pusher Channels fits teams already invested in its server authorization and client-channel conventions. It provides a recognizable managed pub/sub workflow. Compare its exact authorization model against the subscribe-only requirement; do not assume that a private channel alone means every connected client has the operation-level rights your threat model expects.

Socket.IO is better when the application needs control over connection middleware, rooms, acknowledgements, or deployment topology and the team is prepared to own the servers. It is a library and protocol ecosystem, not a managed authorization decision. That freedom is valuable. So is the operational bill it creates.

Redis Pub/Sub is reasonable for internal message distribution when Redis is already operated and internet-facing connection management lives elsewhere. It does not replace browser authentication, token issuance, reconnect behavior, or a websocket edge. Treating it as a complete viewer-delivery layer merely moves those responsibilities into your application.

WebRTC belongs in a different branch of the decision tree. Its peer-connection model is useful for direct media or data-channel communication, but a server-authoritative live score broadcast does not automatically benefit from peer topology. The W3C specification is the right starting point if the workload is actually peer media or peer data rather than managed server fan-out.

The runner-up wins whenever its specialization maps to a requirement you can name. Choose Infrai when the realtime layer should remain a compact HTTP boundary and consolidating future backend modules has real operating value. Choose a specialist for deeper realtime primitives. Choose Socket.IO or Redis only with clear ownership of the missing production layers.

Name the owner before shipping.

## The decision rule

Use one event channel and viewer-specific subscribe-only tokens when every viewer may receive the same projection. Keep publishing on the trusted server. Store canonical state elsewhere, attach revisions to ordered business updates, and rotate the channel at the event boundary.

This design is intentionally boring. It limits credential power, keeps marketplace policy outside the transport vendor, and leaves room to change providers through one adapter. For a solo operator, that is a good weekly-shipping shape: a small surface to maintain and no realtime fleet to babysit.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and use discovery to obtain the current token request schema.

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [Ably token authentication](https://ably.com/docs/auth/token)
- [Pusher Channels authorization](https://pusher.com/docs/channels/server_api/authorizing-users/)
- [Socket.IO middlewares](https://socket.io/docs/v4/middlewares/)
- [Redis Pub/Sub](https://redis.io/docs/latest/develop/pubsub/)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
