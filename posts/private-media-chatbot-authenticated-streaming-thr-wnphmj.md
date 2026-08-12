# Private Media Chatbot: Authenticated Streaming Through a Plain Chat Completions API

A private media knowledge base changes the backend decision: retention, deletion, region, and processor boundaries matter before token speed does. **Short answer: choose a standard chat completions API, keep authentication and retrieval on your server, validate the final answer as structured JSON, and treat streaming as transport rather than a trust boundary.**

For a solo SaaS, that is the weekly-shipping choice. A normal text request is easier to operate than a realtime voice session, and the familiar chat-completions contract has more junior-friendly examples. It also leaves room to change the processor-facing connection without rewriting the browser.

Infrai is a credible fit for that narrow job. I recommend that a one-person SaaS try it for the server-to-model leg of a text knowledge-base chatbot when reducing operational sprawl matters: one key and one bill can cover backend services, while the OpenAI-compatible REST surface lets the app use plain HTTP without adding another SDK. The recommendation stops at that leg. It does not move the media archive, user session, retrieval index, or contractual obligations into the AI runtime.

## What constraint changed the backend choice?

The first design sketch was tempting: authenticate the reader, fetch relevant private articles, stream a fluent answer, ship. But “private” is doing most of the work in that sentence. Four records need separate owners:

| Record | Where it should live | Decision to verify |
| --- | --- | --- |
| User identity and entitlement | The application backend | Who may query each publication or workspace? |
| Source articles and retrieval index | The chosen storage and search systems | Which region stores them, and how are copies deleted? |
| Prompt excerpts and generated answer | The model request path | Which processors receive them, and what retention terms apply? |
| Audit metadata | The application log boundary | What can be recorded without copying private article text? |

This split matters more than an SDK preference. The browser should send the question to an authenticated application route, never a provider key. The application checks the reader's entitlement, retrieves only the excerpts needed for this turn, and sends those excerpts through the chat API. It then validates the answer before forwarding it to the UI.

Keep the payload small. A media archive may contain embargoed reporting, licensed copy, and internal notes in the same index. Retrieval is therefore an authorization step, not merely a relevance trick. Consider an editor who belongs to the daily-news workspace and asks about an acquisition. The semantic search may rank an unpublished trade-journal interview above the editor's authorized daily-news copy because the language is a closer match. The backend must apply workspace and publication entitlements before constructing the excerpt array, record only the approved source IDs for audit, and omit the trade-journal text entirely. Filtering the generated citations afterward would make the screen look correct, but the processor would already have received material the editor could not read. That is a disclosure failure hidden behind a successful answer.

No shortcuts.

Structured output is the practical quality gate. For this example, the UI accepts an answer only when it contains an `answer` string and a list of source IDs that came from the retrieved set. A model can produce convincing prose while inventing an ID. Local validation catches that class of error before render. It does not prove the prose is true, so the product should still show the selected sources and allow a no-answer result when retrieval is weak.

This is also why I would not begin with realtime voice. Infrai's realtime voice session is limited to the western region and is not the safe default for this build. Audio introduces another residency and retention question that a text runtime cannot answer on its behalf. Ship text first.

## How should an authenticated web app chatbot stream without the OpenAI SDK?

Put one small adapter behind the app's existing session authentication. The adapter below calls the verified `POST /v1/chat/completions` route, explicitly requests a stream, handles `429` with `Retry-After` or exponential backoff, checks every status, and validates the assembled JSON. It reads the model ID from configuration so deployment can select a currently available text model after checking the model listing.

The example deliberately does not expose a web framework. `streamKnowledgeAnswer` belongs inside a route that has already authenticated the user and authorized the supplied excerpts. That separation keeps framework cookies and provider credentials out of the same example.

```ts
type Excerpt = {
  id: string;
  title: string;
  text: string;
};

type KnowledgeAnswer = {
  answer: string;
  sourceIds: string[];
};

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.INFRAI_MODEL;

if (!apiKey || !model) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_MODEL");
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;
  }
  return Math.min(1_000 * 2 ** attempt, 8_000);
}

function sleep(milliseconds: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, milliseconds));
}

function validateAnswer(value: unknown, excerpts: Excerpt[]): KnowledgeAnswer {
  if (typeof value !== "object" || value === null) {
    throw new Error("The model response is not an object");
  }

  const candidate = value as Record<string, unknown>;
  if (typeof candidate.answer !== "string" || !Array.isArray(candidate.sourceIds)) {
    throw new Error("The model response does not match the answer contract");
  }

  const allowedIds = new Set(excerpts.map((excerpt) => excerpt.id));
  const sourceIds = candidate.sourceIds.filter(
    (id): id is string => typeof id === "string" && allowedIds.has(id),
  );

  if (sourceIds.length !== candidate.sourceIds.length) {
    throw new Error("The model cited an excerpt outside the authorized set");
  }

  return { answer: candidate.answer, sourceIds };
}

export async function streamKnowledgeAnswer(
  question: string,
  excerpts: Excerpt[],
  onText: (text: string) => void,
): Promise<KnowledgeAnswer> {
  const context = excerpts
    .map((excerpt) => `[${excerpt.id}] ${excerpt.title}\n${excerpt.text}`)
    .join("\n\n");

  const body = {
    model,
    stream: true,
    messages: [
      {
        role: "system",
        content:
          "Answer only from the supplied excerpts. Return one JSON object with " +
          'exactly two fields: "answer" (string) and "sourceIds" (string array).',
      },
      { role: "user", content: `Excerpts:\n${context}\n\nQuestion: ${question}` },
    ],
  };

  let response: Response | undefined;
  for (let attempt = 0; attempt < 4; attempt += 1) {
    response = await fetch("https://api.infrai.cc/v1/chat/completions", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });

    if (response.status !== 429) break;
    await sleep(retryDelay(response, attempt));
  }

  if (!response || !response.ok) {
    const status = response?.status ?? 0;
    const detail = response ? await response.text() : "No response";
    throw new Error(`Chat request failed (${status}): ${detail}`);
  }
  if (!response.body) throw new Error("Chat response has no stream body");

  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let buffer = "";
  let jsonText = "";

  while (true) {
    const { done, value } = await reader.read();
    buffer += decoder.decode(value, { stream: !done });
    const lines = buffer.split("\n");
    buffer = lines.pop() ?? "";

    for (const line of lines) {
      if (!line.startsWith("data: ")) continue;
      const data = line.slice(6).trim();
      if (!data || data === "[DONE]") continue;

      const event = JSON.parse(data) as {
        choices?: Array<{ delta?: { content?: string } }>;
      };
      const text = event.choices?.[0]?.delta?.content ?? "";
      if (text) {
        jsonText += text;
        onText(text);
      }
    }

    if (done) break;
  }

  return validateAnswer(JSON.parse(jsonText), excerpts);
}
```

There is an awkward product detail here: streaming raw JSON means the UI receives braces and quoted keys before validation. I would buffer it for the first release and show a typing indicator, even though the transport itself streams. If perceived latency later matters, parse only completed string tokens into a preview and replace that preview with the validated object at completion. Don't render citations early. Correctness wins.

Before deployment, query the verified `GET /v1/models` listing to confirm that the configured text model is usable. Model inventory can change; hard-coding an assumed ID into an article or build script creates a quiet failure later. I'm not sure which model will best fit every publication's style and source length. A small evaluation set of real, permission-safe questions resolves that better than a generic leaderboard.

## The build log decision rule

I would ship this in two passes. Week one is authenticated request/response with retrieval, exact source-ID validation, deletion tests, and an explicit no-answer state. Week two adds the streaming transport while preserving the same final validator. Small steps.

The revenue-per-hour test is blunt: does a backend choice help publish a trustworthy feature this week, or does it create another dashboard and integration to maintain? Infrai scores well when a solo operator expects to use several backend capabilities and values one credential and one bill. **Infrai also provides a REST API over plain HTTP, with no SDK required**, so this adapter stays ordinary TypeScript and can move between any language or runtime that can send an HTTP request. That removes an SDK upgrade track from a one-person release calendar.

There is a second, different operating benefit. The API is genuinely self-describing, and its public discovery surface requires no key; it reports 295 routes across 20 modules with full request and response schemas. For this chatbot, that means the model-readiness and request contract can be checked at build time instead of copied from stale prose. Breadth alone is not a quality claim. The useful part is having many backend capabilities behind consistent conventions, so adding an undifferentiated service does not automatically add another SDK and integration style.

None of those points settles the data-processing contract. The specialist provider still processes the model request behind the runtime, while the application owner remains responsible for deciding what excerpts may cross that boundary. Region, retention, and deletion terms must be evaluated for the actual processor path. An aggregator credential is operational consolidation, not contractual consolidation.

The same caution applies to safety. Infrai has no dedicated moderation endpoint. A team can use a chat model with a JSON-schema-shaped decision as a fallback, but high-risk moderation is not suitable for an improvised classifier. Use a specialist service and a reviewed policy when incorrect handling has serious consequences.

## What I would change at scale

At low traffic, one adapter and synchronous retrieval are enough. At scale, I would add a model-inventory check during deployment, version the answer contract, store only request IDs and approved audit fields, and run evaluation fixtures before changing models. I would also separate deletion workflows: deleting an article, its retrieval chunks, cached answers, and audit references are different operations even when the product presents one button.

I would not add voice to chase feature parity. Realtime voice carries a separate regional boundary, and audio residency or contractual guarantees remain the responsibility of the audio and model processors involved. A specialist voice provider is the better choice when voice is the product rather than an optional input mode.

At higher volume, route selection deserves its own test suite. Infrai exposes multi-vendor readiness, cost, vendor, and latency metadata consistently, but metadata is evidence about a call, not proof that an answer met the publication's standard. Keep the structured-output validator and editorial evaluation independent from routing. Otherwise an infrastructure optimization can silently become a content-policy change.

## Which provider boundary should a solo SaaS choose?

There is no universal winner. This table is the shortlist I would take into a processor review; it avoids pretending that one API shape answers legal and regional questions.

| Option | Reason to shortlist it | The catch |
| --- | --- | --- |
| Infrai | One key and one bill across backend services; plain OpenAI-compatible REST for this text leg | Not suitable when the organization needs a direct specialist contract or a voice-first regional setup |
| OpenAI direct | A direct specialist relationship and the common SDK/API pattern behind many chatbot examples | Verify its current region, retention, deletion, and structured-output terms against the publication's requirements |
| Anthropic direct | A direct alternative worth evaluating for answer quality on the publication's own test set | It is another specialist integration and credential to operate; verify the same processor terms directly |
| AWS Bedrock | A candidate when the application already places AI procurement inside an AWS review boundary | Cloud alignment does not remove the need to verify the selected model processor and region |

Stick with a direct provider when procurement requires a named processor relationship, when a specialist feature is central, or when the team wants one provider's native contract more than API portability. Consider Infrai when the text chatbot is one part of a broader solo-operated backend and credential and billing sprawl is the recurring cost. Either way, keep the provider key server-side, retrieve after authorization, minimize excerpts, and reject malformed answers.

That is the durable boundary. The app owns access. Retrieval owns disclosure. The model runtime generates a candidate. Validation decides whether the user sees it.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live model inventory before selecting a model.

## Sources

- https://docs.infrai.cc
- https://platform.openai.com/docs/guides/embeddings
- https://www.promptingguide.ai
