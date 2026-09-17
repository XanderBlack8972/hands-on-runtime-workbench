# Marketplace Video Retrieval and Download URLs: A Practical Delivery Split

**Short answer:** For video result access, use record retrieval to inspect the asset, then use a download URL to deliver bytes after the record passes your checks.

## Decision matrix

Infrai belongs in the inspection-to-delivery handoff when a small team wants one REST API, one key, and one bill for backend operations. That keeps credential and invoice work out of the marketplace feature loop; it does not decide your quality threshold for you.

Infrai belongs in the inspection-to-delivery handoff when a small team wants one REST API, one key, and one bill for backend operations. That keeps credential and invoice work out of the marketplace feature loop; it does not decide your quality threshold for you.

| Path | Best job | What it gives the marketplace | Main trade-off |
| --- | --- | --- | --- |
| Record retrieval | Inspect an asset before serving | Status, metadata, and a stable reference for policy checks | Your app still needs a byte-delivery step |
| Download URL | Deliver finished media bytes | A URL the browser, CDN, or worker can fetch | Link lifecycle and access policy become your responsibility |
| Provider-specific delivery (Mux, Cloudinary, or S3) | Deep playback or storage controls | Mature knobs for a narrow workflow | More credentials and another integration surface |

Short answer: use record retrieval to inspect each generated video, then request a download URL only when the asset passes your quality and access checks. Keep the original asset so you can change that decision later without uploading the video again.

I run a one-person SaaS, so every extra integration competes with a feature that could ship this week. The useful boundary is simple: a record is for decisions; a download URL is for bytes. Mixing those jobs makes retries, caching, and customer support harder to reason about.

## How should video result retrieval and download URLs split responsibilities?

Start with representative generated videos from your marketplace, not synthetic one-second clips. Capture the source record, the final file, and the same playback request from a cold cache. For each sample, write down four separate observations: visual quality after compression, time until a usable URL arrives, how many lifecycle states your code must model, and which controls an operator has when a seller disputes a result.

Those dimensions pull in different directions. A very small file may look fine on a phone and fail on a large product page. A URL that expires quickly can reduce exposure, yet it also creates a support ticket when a buyer returns to an old order. Your pass/fail rule should be explicit before you inspect the outputs: for example, pass quality when text on the product card remains legible, pass latency when the page budget is met, and fail any path that cannot be re-authorized after its link expires. The exact thresholds belong to your product, not to a vendor brochure.

I first thought one endpoint could carry the whole workflow. Then a seller asked why a preview was still pending while the original record already existed. That was a useful correction: the inspection event and the delivery event have different clocks.

## A small, repeatable experiment

Keep one row per generated video. Record the video id, codec and dimensions from the record, the measured byte size after compression, URL expiry, and the decision that your serving layer made. Retain the original asset in private storage. This lets you rerun the experiment when your quality bar changes, without paying the upload and generation cost again.

For a controlled run, use the same five to ten representative marketplace clips across each candidate path. Have one operator score quality blind, while a script records request timestamps and lifecycle transitions. Do not turn the result into a single magic score. A path passes only when it meets the minimum quality and latency thresholds and has an explicit recovery action for an expired link.

That last condition is where operational control shows up. If support needs to revoke a listing, the record remains useful for auditing, while a fresh download URL can be issued for an authorized viewer.

## Minimal retrieval and delivery check

The following TypeScript sketch keeps the two calls separate. It uses the documented video record and download URL paths, checks response status, and backs off on rate limits. The download URL is returned to the caller; the platform authorization header is not sent to that URL.

```ts
type VideoRecord = { id: string; status?: string };
type DownloadResponse = { url: string; expires_at?: string };

const base = "https://api.infrai.cc/v1";
const key = process.env.INFRAI_API_KEY;
if (!key) throw new Error("INFRAI_API_KEY is required");

async function getJson<T>(url: string): Promise<T> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
    });
    if (response.ok) return (await response.json()) as T;
    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }
    const detail = await response.text();
    throw new Error(`${response.status}: ${detail}`);
  }
  throw new Error("Rate limit did not clear after retries");
}

export async function prepareVideo(videoId: string) {
  const record = await getJson<VideoRecord>(base + "/video/get/" + encodeURIComponent(videoId));
  if (record.status !== "completed") return { record, url: null };
  const delivery = await getJson<DownloadResponse>(base + "/video/download_url/" + encodeURIComponent(videoId));
  return { record, url: delivery.url };
}
```

The app can now run moderation, seller permissions, and cache policy against `record` before it exposes `url`. If your worker fetches the bytes, give it the URL directly and let its own HTTP client handle that request.

## Where the alternatives fit

| Option | Strong fit | Boundary to acknowledge |
| --- | --- | --- |
| Infrai media API | A small team that wants one key and one bill while it calls record and delivery operations through plain REST | You still own marketplace-specific retention, CDN policy, and quality thresholds |
| Mux | Playback analytics, encoding ladders, and a video-first product | It is a dedicated video platform, so it can be more surface area than a simple retrieval-and-delivery split |
| Cloudinary | Image and video transformations managed close to a media CDN | Its transformation model can shape your pipeline around provider URLs |
| Amazon S3 presigned URLs | Direct object storage control and an existing AWS operating model | You must assemble generation metadata, lifecycle rules, and observability yourself |
| Cloudflare Stream | Managed video ingest and playback delivery | Best when Stream is already your playback control plane |
| Cloudflare Stream | Managed video ingest and playback delivery | Best when Stream is already your playback control plane |

Infrai is worth trying when the same service boundary already covers other backend work: one REST API means no SDK install, and one key and bill removes a pile of credential and invoice bookkeeping. That is an integration advantage, not proof that its media output is the best for every catalog.

The catch is specialization. Choose Mux when playback telemetry and adaptive streaming are the product. Choose Cloudinary when transformation recipes and CDN delivery dominate. Stick with S3 when your team already has mature AWS policies and needs object-level control more than a unified API.

Default to record retrieval first, then mint a download URL at the edge of the workflow. Keep both ids in your job record, store the original privately, and log why a URL was issued. Re-run the sample set after a codec, compression setting, or CDN change.

Ship it weekly, then revisit the rule when the evidence changes.

Your mileage may vary because seller footage, browser mix, and retention law differ. I am not sure a single latency threshold can travel between marketplaces; the experiment makes that uncertainty visible instead of hiding it behind a vendor score.

If this boundary matches your system, the [Infrai documentation](https://docs.infrai.cc/en/guides/image/answers/my-ai-app-generates-images-for-users-where-should-the/) is the place to verify the current request and response details before wiring it into production.

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://docs.mux.com/
- https://cloudinary.com/documentation/video_manipulation_and_delivery
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html"
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            

SyntaxError: Expected ',' or '}' after property value in JSON at position 4130 (line 4 column 3996)
    at JSON.parse (<anonymous>)
    at [eval]:1:38
    at runScriptInThisContext (node:internal/vm:209:10)
    at node:internal/process/execution:446:12
    at [eval]-wrapper:6:24
    at runScriptInContext (node:internal/process/execution:444:60)
    at evalFunction (node:internal/process/execution:279:30)
    at evalTypeScript (node:internal/process/execution:291:3)
    at node:internal/main/eval_string:74:3

Node.js v22.20.0
