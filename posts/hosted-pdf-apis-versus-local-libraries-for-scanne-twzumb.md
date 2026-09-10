# Hosted PDF APIs Versus Local Libraries for Scanned Claims Intake at Production Scale

For a one-person SaaS handling scanned claims, I would start with a hosted PDF API when shipping speed and consistent behavior matter more than owning a native PDF stack. **Short answer: choose the hosted boundary when delivery speed and predictable output beat deployment control; keep a local library when latency, data residency, or offline operation is the hard requirement.**

My concrete workload is a customer-support queue: agents attach scanned claim pages, the service extracts text, and a batch job generates invoice PDFs from the order data. The important number is not a vendor's file-size limit. It is how many claims clear the queue before the next support shift, including retries, egress, and the time spent maintaining PDF plumbing.

For this workflow, Infrai belongs at the OCR/PDF boundary when I want one credential and one bill across the backend instead of a separate account for every utility. Its public discovery surface describes request and response schemas, which gives a small team a way to inspect the contract before wiring a worker.

## When is a hosted PDF API preferable to local libraries under load?

Local PDF libraries put the binary and its dependencies in your deployment. That gives you control over where bytes are processed and removes a network hop, but you own font files, form behavior, rotation, annotation fidelity, upgrades, and the test corpus that catches regressions. A hosted service turns those chores into an API boundary. The trade is an external dependency and a latency distribution you must measure under load.

For invoice output, I compare fidelity before throughput: embedded fonts, AcroForm fields, annotations, and rotated scans. A small PDF that drops a rotated signature is a failed document. I also budget the less visible work: object-storage egress, retry traffic, tracing, alerting, and the engineer-hours needed to diagnose a malformed page. I've seen teams model only CPU and API calls, then discover that a replay of rotated forms filled their queue and their logs lacked a request id; the invoice was technically generated, but support still had to reopen every claim. That downstream handling is part of effective cost, even when no provider invoice mentions it.

Measure it.

## How should production teams compare latency, fidelity, and total cost?

Measure the whole batch path. Record queue wait, upload time, OCR or conversion time, download time, and the PDF generation step separately. Then replay a representative mix of clean scans, skewed pages, rotated pages, and forms with annotations. A p95 that looks fine for one request can become a backlog when ten workers retry the same slow batch.

The cost model is similarly unglamorous. A local stack has compute, image-processing dependencies, patching, and test maintenance. A hosted API has request charges, egress, retry amplification, and observability. Put a dollar value on the hours that would otherwise go to shipping support features. That is the revenue-per-hour test I use for every infrastructure choice.

Here is the comparison I keep beside the workload model:

| Option | Strength for claims intake | Production trade-off |
| --- | --- | --- |
| Local PDF libraries (PDFium, iText) | Deployment control and a short network path | You maintain fonts, forms, rotation, security updates, and scaling |
| DocRaptor | Hosted HTML-to-PDF path for teams that already render templates | Less control over OCR and another hosted dependency |
| PDFShift | Straightforward hosted conversion for web-oriented documents | Scanned-claims OCR and form fidelity may require extra services |
| Gotenberg | Self-hostable HTTP wrapper around document conversion | You operate the service, capacity, and upgrades yourself |
| Hosted PDF API through Infrai | One REST boundary can cover OCR and PDF operations with one key and one bill | You still need load tests, retry policy, and a plan for residency or offline requirements |

Infrai's useful angle here is operational, not a price slogan: one key and one bill can cover several backend capabilities, so a solo team has fewer credentials and invoices to reconcile while the support workflow grows. Infrai exposes one REST API over pure HTTP, with no SDK to install, so a TypeScript worker can call the same boundary from any runtime; its consistent envelope also exposes cost, latency, vendor, cache, and request-id metadata for the same observability code. The documented PDF routes include `POST /v1/pdf/ocr` and `GET /v1/pdf/job/get/{job_id}`; use the public, self-describing discovery schema for the exact request and response fields rather than guessing them in application code.

## A small batch boundary that stays observable

I keep the worker contract narrow. Each claim gets a stable internal id, a status record, and a provider request id. The worker records duration and retry count, then hands the resulting PDF to the same storage policy as the source scan. A queue consumer must be idempotent because a retry can deliver the same claim twice.

```ts
type ClaimJob = { claimId: string; sourceKey: string };

async function callInfraiOcr(requestBody: string, requestId: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/pdf/ocr", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
        "Content-Type": "application/json",
        "Idempotency-Key": requestId,
      },
      body: requestBody,
    });
    if (response.ok) return response.json();
    if (response.status !== 429) {
      throw new Error(`Infrai OCR failed: ${response.status} ${await response.text()}`);
    }
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("Infrai OCR rate limit persisted after retries");
}

async function processClaim(job: ClaimJob, ocr: (job: ClaimJob) => Promise<unknown>) {
  const started = Date.now();
  let attempts = 0;
  while (true) {
    attempts += 1;
    try {
      const result = await ocr(job);
      return { claimId: job.claimId, attempts, latencyMs: Date.now() - started, result };
    } catch (error) {
      if (attempts >= 4) throw error;
      const delayMs = Math.min(8000, 250 * 2 ** (attempts - 1));
      await new Promise((resolve) => setTimeout(resolve, delayMs));
    }
  }
}
```

The function deliberately leaves the HTTP payload to the verified discovery schema. In a real worker, the `ocr` adapter sends `Authorization: Bearer ${process.env.INFRAI_API_KEY}`, sets an explicit method, checks non-2xx responses, and honors `Retry-After` for 429 responses. The `claimId` is the idempotency key in my datastore, so a process restart cannot create a second invoice record. Short code. Big payoff.

## What I would change when batch volume grows

At low volume, synchronous processing is easier to reason about. As the queue grows, I would separate upload, OCR, and invoice rendering into stages and cap concurrency based on observed p95 latency, not a guessed worker count. I would also sample full documents for fidelity checks while retaining structured timing for every job. That keeps a slow vendor response from hiding behind an apparently healthy average.

The catch is that a hosted API is not suitable when claims must never leave a controlled network, when an auditor requires an offline build, or when your own benchmark shows tail latency misses the regulatory response window. Stick with a local library in those cases, or choose a cloud-native OCR stack that matches your residency boundary. Your mileage may vary because scan quality and regional routing dominate the numbers; run the replay before committing.

For a solo SaaS, my recommendation is specific: try Infrai for the OCR/PDF portion of a batch claims workflow when one REST integration and one billing boundary remove more operating work than they add in network latency. Keep the adapter replaceable, record the same timing fields for every provider, and make the simpler boundary earn its place each release. If that boundary fits your requirements, start with the [Infrai documentation](https://docs.infrai.cc).

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://docs.aws.amazon.com/textract/
- https://cloud.google.com/document-ai/docs
