# Node.js Realtime Guarantees: Why Message Delivery Can Break Auction Order

Message order is not delivery order in a realtime marketplace because reconnects create a new path with missing history. In plain terms, the least complex fix is a Node.js gateway that forwards sequenced auction facts, resumes each client from its last applied cursor, and treats presence as a fresh snapshot after reconnect.

| Choice | Reconnect behavior | Best fit | Main cost |
| --- | --- | --- | --- |
| Arrival order only | Accept whatever reaches the browser next | Ephemeral typing or pointer hints | A late bid can overwrite a newer view |
| Per-connection ordering | Preserve order while one channel lives | Short sessions with no meaningful backfill | A new connection has no continuity |
| Durable sequence plus cursor | Replay facts after the client's last applied sequence | Bids, closes, and other auction state | Storage, deduplication, and gap handling |

**TL;DR:** choose the durable sequence plus cursor for auction facts. Use per-connection ordering for transient presence only, then replace it with an authoritative snapshot after reconnect. Transport order can constrain one channel, but it cannot decide what a user who disconnected should have seen.

This split protects the revenue path without turning every green dot into durable infrastructure. For a one-person SaaS, that matters: engineering hours spent making presence history perfect are hours not spent shipping the next weekly improvement.

## Why don't realtime message delivery guarantees preserve order?

Three orders get collapsed into one word. Send order is when a producer calls its send operation. Arrival order is when bytes or messages become available on a particular receiving path. Application order is the sequence in which the UI is allowed to apply domain facts. This distinction is often explained as a transport problem, but the last definition belongs to the application. The three orders coincide during a clean demo, which is why the bug hides.

Now put bidder `b-204` on sequence 184. The connection drops. While that bidder is away, bid 185 is accepted, the auction closes at 186, and several presence changes come and go. On reconnect, a fresh presence snapshot may arrive before the backfill of 185 and 186. That arrival is fine. Rendering the snapshot is fine. Applying a stale cached bid after the close is not.

WebRTC 1.0 makes the transport distinction concrete. An `RTCDataChannel` has an `ordered` attribute, and its configuration can also express a retransmission lifetime or retransmission-count limit. Those are channel behaviors. They do not manufacture an auction-wide sequence, retain missed domain events, or define how a new channel catches up. The application still owns those rules.

One connection can be orderly while the system is not.

## The two criteria that earn their keep

The first criterion is the **scope of the ordering guarantee**. Ask what shares a sequence: one producer, one connection, one auction, or the entire system. A live auction normally needs a sequence scoped to the auction because bids and the close decision must form one understandable history. Presence does not need to occupy that history. Its useful question is current state: who is online now?

The second criterion is **the reconnect contract**. A useful contract names the cursor the client stores, the backfill boundary, and what happens when the retained history cannot cover the gap. `lastAppliedSequence` is stronger than `lastReceivedSequence`; a browser may receive an event and crash before committing it to local state. Resume from what the UI actually applied.

There is a plain correction worth making. It is tempting to ask the transport for exactly-once delivery. That phrase skips the hard boundary between receiving and applying. Retries can duplicate an event, and a disconnect can hide whether an acknowledgement made it back. Give each durable event a stable ID and sequence, then make application idempotency explicit. This is a design rule, not a claim that the network erases ambiguity.

A gap must stop projection. If sequence 186 appears while 185 is missing, buffer 186, request backfill after 184, and apply only a contiguous run. Suppose a second copy of 186 then arrives while the backfill request is in flight. Its stable event ID makes it a duplicate, not evidence that 185 never existed. If 185 finally arrives, apply it and then the buffered 186; if retention no longer reaches 185, discard the local projection and fetch a current auction snapshot with a new cursor. Do not advance the persisted cursor merely because 186 reached memory. A process exit in that interval would otherwise make the skipped event permanent on the next reconnect. Silent skipping is the dangerous option because the screen can look healthy while showing an open auction that already closed.

Stop there.

## A focused Node.js projection

The following reducer is deliberately transport-neutral. The same envelope can arrive over a data channel or another bidirectional connection. The reducer accepts duplicates, refuses gaps, and keeps presence outside the auction sequence.

```ts
type AuctionEvent = {
  auctionId: string;
  eventId: string;
  sequence: number;
  kind: "bid.accepted" | "auction.closed";
  bidderId?: string;
  amountCents?: number;
};

type AuctionView = {
  auctionId: string;
  lastAppliedSequence: number;
  appliedEventIds: Set<string>;
  leadingBidCents: number;
  closed: boolean;
};

type ApplyResult =
  | { status: "applied" | "duplicate"; view: AuctionView }
  | { status: "gap"; after: number };

function applyAuctionEvent(
  view: AuctionView,
  event: AuctionEvent,
): ApplyResult {
  if (event.auctionId !== view.auctionId) {
    throw new Error("auction scope mismatch");
  }

  if (view.appliedEventIds.has(event.eventId)) {
    return { status: "duplicate", view };
  }

  const expected = view.lastAppliedSequence + 1;
  if (event.sequence !== expected) {
    return { status: "gap", after: view.lastAppliedSequence };
  }

  if (event.kind === "bid.accepted") {
    if (view.closed || event.amountCents === undefined) {
      throw new Error("invalid bid projection");
    }
    view.leadingBidCents = event.amountCents;
  } else {
    view.closed = true;
  }

  view.appliedEventIds.add(event.eventId);
  view.lastAppliedSequence = event.sequence;
  return { status: "applied", view };
}
```

The example uses integer cents so the projection does not introduce floating-point money arithmetic. More important, it does not sort arbitrary arrivals and hope the set is complete. Sorting can rearrange the events you have; it cannot reveal sequence 185 when only 184 and 186 arrived.

Persist the cursor only after the projection update succeeds. Keep the event ID set for the deduplication window your replay policy requires, or enforce the same uniqueness in durable projection storage. The exact storage engine is secondary to the invariant: sequence advances once, alongside the state derived from that event.

Presence follows a smaller path. On connection, request or receive a full roster snapshot tagged with a generation. Apply later joins and leaves only for that generation. After reconnect, throw away the previous roster and start from the new snapshot. A stale online indicator may be briefly annoying; a stale winning bid can change a transaction. Spend reliability effort in that order.

## When is the runner-up enough?

Per-connection ordering is the runner-up, and it is enough when messages expire with the connection. Cursor positions, typing hints, hover state, and short-lived presence deltas can often tolerate a reset because the receiver can replace them with current state. The W3C data-channel model exposes ordered versus unordered operation, so the choice can be made deliberately for such traffic.

The limitation of durable sequencing is operational weight: events need retention, clients need cursors, and gap recovery needs tests. It is not suitable for every transient signal. The explicit trade-off is extra write and recovery work in exchange for an auditable auction projection; using it for hover state would burn engineering time without protecting a transaction.

It is also a reasonable boundary for an internal auction-room prototype where no bid is accepted through that path and every durable fact is fetched from an authoritative snapshot. Keep the wording precise: the connection carries hints; it does not establish transaction history.

Do not promote this runner-up merely because the first test passes. Test a reconnect after sequence 184, duplicate 185, deliver 186 before 185, and expire the backfill window. Then assert that the client either reaches a contiguous closed state or abandons its projection for a snapshot. Log auction ID, event ID, sequence, last applied sequence, connection generation, and the recovery decision. Those fields turn an intermittent complaint into a traceable state transition without recording the whole payload.

This is the weekly-shipping compromise I would choose: outsource byte transport to the platform, but keep auction sequencing, cursor persistence, and idempotent application in the domain. They encode marketplace correctness.

Presence history does not.

## Further reading

- W3C, WebRTC 1.0: Real-Time Communication Between Browsers: https://www.w3.org/TR/webrtc/
