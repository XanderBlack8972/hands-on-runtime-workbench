# How to Integrate Text-to-Image for Course Catalogs in 4 Stages: Quality Before Latency

TL;DR: For an education catalog, generate a draft image quickly, but publish only after deterministic checks pass. Keep the text-to-image call behind a small TypeScript interface, record the prompt and model response metadata, and send slow or ambiguous jobs to a batch queue. The useful unit is not one generated picture. It is a catalog item that has a usable image, a traceable decision, and a predictable retry path.

| Path | Best use | Main risk |
| --- | --- | --- |
| Synchronous generation | Preview while an editor is working | A slow request blocks the editor |
| Asynchronous queue | Publishing a small set of approved items | More state to observe and retry |
| Batch processing | Overnight enrichment of a large backlog | Results arrive later and need reconciliation |

My default is the middle path: synchronous previews, asynchronous publication. It keeps the human workflow responsive without allowing an unverified image into the student-facing catalog. Keep the first path cheap to abandon.

## What does quality mean for a course catalog image?

A vague prompt such as `a fun math course` can produce an attractive image and still fail the job. A catalog image has to support recognition, avoid misleading claims, and remain consistent with the course description. The first quality gate is therefore semantic, not artistic.

I split the input into fields before generating anything: subject, learner level, format, accessibility constraints, and prohibited claims. A messy description might say: “Hands-on fractions for ages 9–11, no prior experience, printable exercises included.” The normalized prompt should preserve those facts while leaving room for visual interpretation. It should not invent a classroom, a credential, or a result the course does not promise.

The second gate is operational. Store a content hash, normalized prompt, request timestamp, output identifier, and validation result. If an editor rejects an image, the next attempt must be explainable. Otherwise every retry becomes a fresh guess, and a one-person SaaS turns into a support queue.

Here is the revision rule I use for the pipeline design. A catalog description is version 17, so its prompt hash is tied to version 17. The generator returns an asset for that revision, and the worker records the result even if the editor is no longer on the page. If the editor changes the age range to a different learner group before the worker finishes, the old result stays a draft and cannot quietly replace the new candidate. The worker compares revisions at the publication boundary, where the decision is cheap to inspect. This also makes retries boring: retry the same revision with the same idempotency key, or deliberately create a new revision and say why. A database row with `generated`, `validated`, `reviewed`, and `published` states is more useful here than a clever prompt template because it tells support what happened without asking an engineer to reconstruct it from logs.

Keep it explicit.

## How should a Node.js text-to-image API integration handle quality?

The application should know what it needs from a generator, not every detail of a provider-specific SDK. In a Node.js integration, this keeps the catalog pipeline testable and leaves the runtime choice as an implementation detail. The API call belongs behind this boundary.

```ts
type ImageRequest = {
  prompt: string;
  catalogId: string;
  idempotencyKey: string;
};

type ImageResult = {
  assetUrl: string;
  requestId: string;
  createdAt: string;
};

interface TextToImage {
  generate(input: ImageRequest): Promise<ImageResult>;
}

export async function createDraft(
  generator: TextToImage,
  input: ImageRequest,
): Promise<ImageResult> {
  if (input.prompt.trim().length < 20) {
    throw new Error("Prompt needs normalized catalog context");
  }

  return generator.generate(input);
}
```

The length check is not a quality model. It is a cheap signal that the normalization step did not silently return an empty field. In production, I would also reject a missing catalog ID and make the idempotency key stable for the same catalog revision. That prevents a browser refresh from creating duplicate work.

A request that finishes in a preview window can be synchronous. A request that crosses that window belongs in a queue. Pick the boundary from editor behavior and measured percentiles, then keep the policy in configuration so it can change without rewriting the content pipeline.

```ts
type Route = "preview" | "publish" | "batch";

export function chooseRoute(
  itemCount: number,
  hasEditorWaiting: boolean,
): Route {
  if (hasEditorWaiting && itemCount === 1) return "preview";
  if (itemCount <= 20) return "publish";
  return "batch";
}
```

The number 20 is a routing example, not a universal limit. A 30-second preview budget is another policy example, not a claim about any particular API. Measure queue age, generation duration, validation failures, and the fraction of jobs that need a second attempt. A low median can hide a painful tail. For a solo operator, tail latency matters because it creates manual status checks at exactly the time reserved for shipping the next feature.

That is the contract.

Batch processing is useful when the catalog is already approved and the result can arrive later. The Batch API documentation describes a file-based asynchronous workflow and a 24-hour completion window; that makes it a scheduling tool, not a suitable replacement for an editor waiting on a preview. The same distinction applies to any asynchronous service: name the state transitions and persist them.

## Stage 3: Validate before the asset becomes visible

Do not treat a successful HTTP response as publication approval. Validate the response shape, fetch the asset through a bounded client, check its media type, and associate it with the exact catalog revision that produced the prompt. Rejecting a result is cheaper than explaining a mismatched image to a teacher.

```ts
type Candidate = {
  catalogRevision: string;
  assetUrl: string;
  requestId: string;
};

export function canPublish(
  candidate: Candidate,
  currentRevision: string,
  reviewPassed: boolean,
): boolean {
  return (
    candidate.catalogRevision === currentRevision &&
    candidate.assetUrl.startsWith("https://") &&
    candidate.requestId.length > 0 &&
    reviewPassed
  );
}
```

This gate is intentionally boring. It does not claim that a URL proves an image is safe or relevant. Those checks need their own evidence: a human review for high-impact catalog changes, automated moderation where appropriate, and a visual or metadata check suited to the asset policy. Keep “generated,” “validated,” “reviewed,” and “published” as separate states.

## When is the slower path the better path?

Use the slower path when the description is ambiguous, the learner age is sensitive, or a wrong visual would change the perceived promise of the course. Use the fast path for a reversible draft that an editor can replace before publication. This is a quality decision with a latency consequence, not a latency decision with quality as an afterthought.

I would track four numbers per catalog revision: time to first preview, time to approved asset, rejection rate, and retry rate. Add the prompt hash and request ID to logs, but avoid putting student data or raw descriptions into ordinary application logs. Alerts should fire on stuck states and rising rejection rates, not on every individual slow request.

The runner-up architecture is fully synchronous generation. It is simpler for a tiny catalog and easier to debug locally. It becomes the wrong default once editors can submit multiple items or leave the page before completion. A fully batch-oriented design has the opposite trade-off: predictable worker load, weaker feedback.

For a one-person product, that boundary is the whole decision. Outsource the undifferentiated image-generation call. Keep normalization, state transitions, validation, and audit history in your code. Those parts explain your catalog behavior to customers and make weekly shipping possible.

## Sources

- https://platform.openai.com/docs/guides/batch
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Status
- https://www.rfc-editor.org/rfc/rfc9110
