# Seller Catalog Imports: Queued Batches vs Streamed Rows for Visible Progress

Short answer: use a queued batch with durable item states when an edtech seller uploads a catalog; use streamed rows only when the user must see each thumbnail almost immediately and can tolerate more moving parts. The choice is about thumbnail quality versus bandwidth, not about making an upload endpoint look busy.

A seller catalog import is a small data pipeline disguised as a form. A row can contain a title, a source image, an accessibility label, and a thumbnail that must work on a slow classroom connection. If the request tries to validate every row, fetch every image, resize it, and return one giant response, the first timeout turns the whole import into a guessing game.

I run a one-person SaaS, so my useful metric is revenue per hour. Every minute spent answering “did row 418 finish?” is a minute not spent shipping a feature. The system needs a boring answer that support, a seller, and a retrying worker can all read.

Ship weekly.

## The decision matrix

| Approach | Best fit | Main cost | Progress signal |
| --- | --- | --- | --- |
| Queued batch | Large catalogs, high-quality thumbnails, resumable work | A worker and a state store | Counts plus per-row status |
| Streamed rows | Small catalogs, immediate previews, short-lived sessions | Connection lifetime and backpressure | Events while the request is open |
| Synchronous request | Tiny test imports only | Timeout and retry amplification | One final response |

For production seller imports, I choose the queued batch. It lets thumbnail generation spend enough bandwidth on the source image without tying that work to a browser connection. The streamed option remains useful for a 20-row preview flow, where seeing the first few results matters more than surviving a laptop sleep.

The catch is operational weight. A queue is another moving part, and a batch can feel slower because the UI waits for a state transition instead of watching a request spin. That is a fair trade only if the import can be resumed and explained.

## What should a batch status model expose?

A useful status model separates the batch from its rows. The batch has `accepted`, `running`, `completed`, `completed_with_errors`, or `cancelled`. Each row has a stable id, an attempt count, a media result, and a terminal reason. “Progress: 62%” is not enough; a seller needs to know whether the remaining 38% is still processing, rejected for an unsupported format, or waiting for a retry.

I keep the event history append-only and derive the summary from it. That makes a duplicate worker visible instead of silently incrementing a counter twice. A row transition should be conditional: `queued -> processing -> succeeded|failed`, with `processing -> queued` allowed only when a lease expires. The worker owns the lease; the browser never does.

For image inputs, store the original object reference and the generated thumbnail reference separately. Do not overwrite the seller's source. A 320-pixel thumbnail can be a good classroom list preview, while the catalog detail view may need a larger derivative. The media format guidance from MDN is a useful baseline for deciding which source formats browsers can decode and which output formats fit the delivery path.

Bandwidth is where the quality decision becomes concrete. Downloading every original at full size can punish sellers on mobile networks, but resizing a tiny source into a large card produces a blurry catalog. Record source dimensions and bytes before processing. Then set a policy: reject an image below the minimum useful dimensions, preserve aspect ratio, and generate one predictable thumbnail size for the first screen. The policy is testable, and it gives product a reason for each rejection.

## How do queued imports make progress observable without lying?

The API should return a batch id after validating the envelope, not after doing the media work. A minimal TypeScript shape keeps the contract independent of a particular queue vendor:

```ts
type BatchState = "accepted" | "running" | "completed" | "completed_with_errors" | "cancelled";
type RowState = "queued" | "processing" | "succeeded" | "failed";

type ImportRow = {
  id: string;
  sourceUrl: string;
  state: RowState;
  attempts: number;
  thumbnailUrl?: string;
  errorCode?: "unsupported_format" | "too_small" | "fetch_timeout" | "decode_failed";
};

type ImportBatch = {
  id: string;
  state: BatchState;
  total: number;
  finished: number;
  succeeded: number;
  failed: number;
  updatedAt: string;
};
```

The browser can poll a batch summary or subscribe to events, but both paths read the same state. That matters during a deploy: a reconnect should not reset the progress bar. I include `updatedAt` and a monotonic event id so the client can ignore stale responses.

A worker should acknowledge a row only after the thumbnail object and its metadata are durable. If the process dies after the image write but before the state update, an idempotency key based on batch id plus row id lets the next attempt reuse the existing derivative. If the state update wins first, the worker must still be able to inspect the object reference. These are ordinary failure windows, not reasons to hide progress.

I once treated a 12,000-row import as a single promise and watched a retry create a second set of thumbnails. The response code was 202, so it looked healthy; the accounting was not. The fix was a unique row key, a lease timeout, and a reconciliation job that compared expected and stored objects. I added a database constraint so a second worker could not claim the same row, then made the object key deterministic instead of embedding a timestamp. The reconciliation pass compared the expected row ids with stored object metadata, marked duplicates for review, and emitted one alert with the batch id. I also changed the UI copy: a retry became a visible row transition rather than a fresh import. It added a few hours of implementation and removed a recurring support ticket. More importantly, the next deploy had a clear recovery story instead of a superstition about never retrying.

## Where does the streamed approach win?

Streaming rows is a good fit for a seller preview screen. The server reads a bounded input, emits a result as each thumbnail is ready, and closes the connection when the small set is done. The UI gets a satisfying first result quickly, and the seller can correct a bad source before submitting the full catalog.

It is a poor fit for an overnight import, a catalog larger than the connection's practical lifetime, or a user who may close the tab. Backpressure also needs an explicit rule: cap in-flight image fetches, stop accepting rows when the queue is full, and report the pause rather than buffering unbounded data in memory.

Choose streaming when immediate feedback is the product requirement. Choose a queue when recovery is the requirement. Mixing both is reasonable: stream the first 20-row validation preview, then submit the accepted rows as a durable batch.

## A launch checklist for the thumbnail path

Before shipping, I test the unpleasant cases, not just the happy upload.

- Submit the same batch twice and verify one logical result.
- Kill a worker after writing a thumbnail and before marking the row complete.
- Let a lease expire and ensure only one replacement worker wins.
- Feed a valid image with an unsupported container, a truncated file, and a source below the minimum dimensions.
- Reconnect the browser halfway through and confirm the counts do not jump backward.
- Measure original bytes, derivative bytes, processing latency, and retry count by batch.

The dashboard should distinguish “no rows finished yet” from “the worker has not heartbeated.” Those states look identical in a single percentage, but they lead to different actions. I also keep a sampled row-level log with the batch id and event id; full image URLs do not belong in logs.

There is no universal thumbnail size or transport. Your mileage may vary with the sellers' source mix and the classroom devices you support; I am not sure a single default can serve both a low-bandwidth phone and a retina desktop without an adaptive policy. Measure those cohorts before changing the queue design.

For a one-person team, this is the practical boundary: outsource the undifferentiated image conversion, but keep ownership of state transitions, idempotency, and the progress language shown to users. Ship the smallest queued path that can explain every row. Then add streaming for the preview workflow if the product actually benefits from it.

## Further reading

- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://www.w3.org/TR/eventsource/
- https://www.rfc-editor.org/rfc/rfc9110
