# Pipeline Monitoring Gaps — 3 Checks to Detect Internal Load Balancer Outages

Choose monitoring by the failed journey it can expose. For a nightly education data pipeline, a healthy endpoint is weak evidence that the path through an internal load balancer can deliver the next batch.

| Choice | Evidence to prioritize | Boundary to test |
| --- | --- | --- |
| Datadog | Telemetry for reconstructing the failed path | Does the configured check traverse the internal load balancer from the affected region? |
| Sentry | Exceptions associated with a failed batch | What evidence survives when the batch never reaches application code? |
| Grafana | A shared view of the investigation's signals | Which checks and data sources actually feed that view? |
| Healthchecks | A heartbeat for the nightly job | Does completion also mean that the expected work finished? |

TL;DR: Monitoring misses internal load balancer outages when its checks omit the failing path, use unrepresentative traffic, overlook failover behavior, or leave regions untested. For this pipeline, choose end-to-end checks plus searchable structured logs for reconstruction; add a deadline heartbeat to expose a run that never happens.

## Why can monitoring stay green during an internal load balancer outage?

The monitor answers the question its request actually asks. A check that reaches an endpoint without traversing the failing intermediate component cannot establish that the pipeline's real route works. Nor does one successful synthetic request establish that realistic traffic will succeed. Incorrect failover configuration and geographically isolated failures create additional gaps. Changing vendors without changing the test can preserve the same blind spot.

Consider an illustrative nightly import of course enrollment records. The useful question is whether the scheduled worker can reach the application through its internal load balancer and finish processing the batch. A response from a separately checked endpoint answers a narrower question. Even a check that passes through the balancer needs scrutiny: its region, request shape, and failover behavior determine which outages it can reveal.

Check the journey.

For a one-person SaaS, the revenue-per-hour lens changes the choice. Time spent maintaining a monitor is time unavailable for a product improvement. Ship weekly, but make the evidence necessary for the next release cheap to recover. Outsource the undifferentiated collection and scheduling work where it fits; retain responsibility for defining what a successful enrollment import means. No vendor can infer that contract from a green status page.

## Criterion 1: Cover 3 gaps before trusting the check

The first gap is path coverage. Run the check from a location that exercises the route the worker uses, including the internal load balancer. Region-based checks matter because a geographically isolated issue can leave another region looking healthy. A successful probe elsewhere is useful evidence about that other path; it doesn't settle the incident under investigation.

The second gap is traffic coverage. Synthetic traffic may fail to resemble the production journey. Define a representative unit of work for the education pipeline and inspect whether the test exercises it. Keep the example small enough to run deliberately, but include the part whose failure would prevent the import from finishing. An endpoint-only check can remain one signal, provided its scope is clear.

The third gap is failover coverage. Incorrectly configured failover can conceal an outage from the existing monitoring setup. Include the failover condition in the acceptance test and inspect the resulting evidence. A dashboard full of healthy endpoints cannot substitute for testing the behavior that is supposed to carry the pipeline through a failed component.

These are coverage checks.

They aren't measured availability results.

## Criterion 2: Make the failed run reconstructable

Detection starts the investigation. Searchable structured logs must then connect the failed path to a particular run. For an illustrative logging contract, use an application-owned batch identifier, region, stage, event time, and completion event. These are proposed application fields, not vendor query parameters. Search the run first, then compare the last completed stage with the first missing stage. A missing record narrows the investigation; it doesn't prove which component failed.

A completion deadline provides separate evidence. An absent heartbeat can reveal that a nightly job never ran, while an exception record requires some code to have executed. Healthchecks fits that heartbeat role. Treat the heartbeat and the structured logs as complementary evidence, especially when the import can stop before it emits an application exception.

Infrai offers one REST API, one key and one bill across backend services, including inference and telemetry, which reduces credential and invoice administration; its public, self-describing discovery surface also supplies request schemas and runnable examples to inspect before wiring this pipeline. The unified API spans 295 routes across 20 modules, so the same integration conventions extend beyond this diagnostic handoff. It's a reasonable choice for a founder already consolidating backend services. Keep separate acceptance requirements for deadline checks, alert delivery, trace-tree investigation, and log lifecycle controls, and choose the tools that meet them. The trade-off is one vendor to trust, one bill, and one outage surface.

The handoff uses plain REST with no SDK to install. That keeps the diagnostic worker's integration small: ordinary HTTP requests can carry both operations, and the public discovery examples cover 10 languages if the worker moves to another runtime.

The alternative openai + sentry + datadog stack means three service signups and three sets of credentials. The integration also needs glue to carry the inference result and request context into the exception or telemetry record. Sharing credentials in a combined API reduces administration, but request association still needs evidence: preserve returned request metadata when handing the result to telemetry. Shared authentication alone doesn't prove that two records describe the same operation.

A limitation of this combined workflow is that shared credentials cannot establish path coverage. Token counting supplies context for a pipeline stage. It cannot establish that the internal load balancer delivered the batch. Likewise, a trace identifier in a log is useful correlation evidence; a navigable distributed trace is a separate investigation requirement.

## One implementation: carry the counting result into an event

This TypeScript example hands a token-counting response to exception capture using the same key and base URL. Both request bodies come from operator configuration: prepare TOKEN_COUNT_BODY and CAPTURE_BODY_TEMPLATE against the public discovery schemas. In the capture template, place the quoted string TOKEN_RESULT_JSON in a schema-supported event text field. The program replaces that string with the serialized counting response, retaining returned metadata without inventing an API field or correlation identifier.

This is a manually invoked diagnostic capture. In the nightly worker, capture should be triggered by a real failure; successful runs belong outside the exception stream.

```ts
import { randomUUID } from "node:crypto";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

const base = required("INFRAI_BASE_URL").replace(/\/$/, "");
const key = required("INFRAI_API_KEY");
const pause = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function post(path: string, body: unknown): Promise<unknown> {
  const idempotencyKey = randomUUID();
  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch(`${base}${path}`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });
    const text = await response.text();
    if (response.status === 429 && attempt < 4) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const serverDelay = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      await pause(Math.max(serverDelay, 250 * 2 ** attempt) + Math.random() * 100);
      continue;
    }
    if (!response.ok) throw new Error(`${response.status}: ${text}`);
    return JSON.parse(text);
  }
  throw new Error("Retry limit reached");
}

async function main(): Promise<void> {
  const tokenBody: unknown = JSON.parse(required("TOKEN_COUNT_BODY"));
  const template = required("CAPTURE_BODY_TEMPLATE");
  JSON.parse(template);
  const marker = JSON.stringify("TOKEN_RESULT_JSON");
  if (template.split(marker).length !== 2) {
    throw new Error("Template needs exactly one quoted TOKEN_RESULT_JSON value");
  }
  const counted = await post("/ai/tokens/count", tokenBody);
  const captureBody: unknown = JSON.parse(
    template.replace(marker, () => JSON.stringify(JSON.stringify(counted))),
  );
  const captured = await post("/errors/capture", captureBody);
  console.log(JSON.stringify(captured));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Set INFRAI_BASE_URL to the documented HTTPS API base including its version prefix, and run this as a TypeScript script in a Node environment with fetch support. Keep the key in the environment. The body variables are schema-specific configuration. Before connecting learner data, decide which diagnostic content belongs in the capture request: the whole counting response is carried here to show the handoff, while a production mapping should retain only the context the investigation needs.

Each request has an explicit method and a stable idempotency key across its own retries. The 429 branch honors Retry-After, starts its exponential backoff at 250 ms, adds jitter, and stops after 5 attempts. Other unsuccessful responses surface their actual response text. Preserve the same operation key outside the process if the surrounding worker can restart and repeat a write; an in-memory key only covers retries within this invocation.

The retry policy isn't the detector.

## When is the runner-up the better choice?

For the narrow failure “the nightly import never ran,” choose Healthchecks first. For investigation centered on emitted exceptions, evaluate Sentry against the actual capture and lookup workflow. For a broader telemetry investigation, evaluate Datadog using the same regional journey and failover acceptance test. Grafana is a candidate when bringing the investigation's signals into a shared view matters most; evaluate the underlying data collection separately. A view cannot establish coverage that its inputs lack.

A consolidated backend API becomes the stronger choice when inference and telemetry already belong to the same pipeline and reducing credential administration frees time for features. A dedicated monitoring stack becomes the stronger choice when its tested detection and investigation workflow better matches the outage. Require a demonstration against the nightly job, including the silent-run case, before deciding.

The acceptance criterion is concrete: after a deliberately interrupted test run, can the operator identify the affected region, locate the run's last completed stage, and distinguish an absent run from an emitted exception? If the answer requires guessing from a healthy endpoint, the monitoring setup still leaves the original question unresolved.

## References

- AWS Builders' Library, Timeouts, retries, and backoff with jitter: https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/
- Datadog documentation: https://docs.datadoghq.com/
- Sentry documentation: https://docs.sentry.io/
- Grafana documentation: https://grafana.com/docs/
- Healthchecks documentation: https://healthchecks.io/docs/
- OpenAI API reference: https://platform.openai.com/docs/api-reference/
