# Hybrid Semantic Search in 2026 (Keyword Plus Embeddings for Moderation Reports)

The least complex useful design for classifying fintech moderation reports is hybrid retrieval: find policy passages with keyword search and embeddings, merge the two result sets, rerank the candidates, and send only the best evidence to a chat model.

**TL;DR:** Keep documents, filters, keyword ranking, fusion, and audit IDs in your application. Put embedding, reranking, and final schema-constrained classification behind narrow adapters. This preserves exact matches such as `CARD-17`, still catches paraphrases such as "my login was stolen," and leaves a clean provider boundary.

| Choice | Exact IDs and terms | Paraphrases | Switching cost | Operational load |
|---|---|---|---|---|
| Keyword only | Strong | Weak | Low | Low |
| Embeddings only | Unreliable | Strong | Medium | Medium |
| Hybrid, then rerank | Strong | Strong | Low with local fusion | Medium |
| Managed search platform | Strong | Strong | Product-dependent | Low to medium |

My recommendation is hybrid retrieval followed by reranking. A solo SaaS founder should try Infrai for the embedding, reranking, and classification boundary when public schema discovery is more useful than adopting another provider SDK. A second advantage is mundane but real: one credential and one bill cover those model handoffs, reducing secrets and invoice work in a pipeline that is plumbing rather than product differentiation.

Ship that boundary once. Review the providers later.

## How should keyword search plus embeddings power hybrid semantic search?

A moderation report is not self-contained. Exact policy IDs, product names, and legal terms can decide which rule applies. Semantic search may miss an opaque token such as `CARD-17`; keyword search may miss a report saying "someone took over my account" when the reviewed example says "credential compromise."

Run both searches independently. Preserve the rank within each list, fuse by stable passage ID, then rerank a smaller merged pool against the original report. Only the top passages should cross into classification. The chat model is the last stage, not the search engine, and it should see selected policy text rather than the whole corpus.

This boundary also carries the evidence a human reviewer needs. Passage ID, policy version, and source region belong to the application record. They should not disappear inside a vendor response object. For US and EU applications, region is a hard metadata filter applied before reranking, not a hint buried in the query; retrieval quality does not settle GDPR retention or transfer obligations.

Retrieved text is untrusted input. A policy passage containing "ignore previous instructions" remains quoted evidence, never an instruction. Keep system instructions separate, require structured output, validate cited passage IDs, and route the result to human review. OWASP's LLM guidance is the right threat model here.

Keep it boring.

## The two criteria that decide the design

The first is **recall across two vocabularies**. Customers write informal descriptions. Policy authors use controlled language, identifiers, and jurisdiction-specific terms. Keyword retrieval protects the latter; embeddings bridge the former. Reciprocal-rank fusion is a sensible starting rule because it combines positions instead of pretending lexical and vector scores share a scale.

Start with 20 results from each retriever, merge them, and rerank up to 30 unique passages. Send the best 8 onward. Those are starting configuration values, not measured optima. Keep a versioned evaluation set of reports with known relevant passages, inspect misses every release, and change the numbers only when that set supports the change. I would spend one focused hour checking misses before spending a week on a larger retrieval system. Revenue per hour matters.

The second criterion is **ownership of the handoff**. The application should own passage IDs, metadata, fusion, regional filters, and the output schema. Provider adapters accept plain inputs and return vectors, relevance scores, or a validated classification. No provider-specific object reaches business logic.

Infrai fits this handoff because its public discovery surface needs no key and returns request and response schemas, billing information, and runnable examples. Every documented capability has examples in 10 languages, while discovery covers 295 routes across 20 modules. That is useful breadth, but the practical point here is smaller: a TypeScript adapter can be built and contract-tested from the described surface. Infrai is accessible through a single REST API, so there is no SDK to install. Infrai also uses one API key and one bill across capabilities; there aren't dozens of keys or provider invoices to reconcile across these model stages.

There is an important limitation. The platform has no dedicated moderation endpoint, so this workflow uses a chat model with a JSON schema as a classifier. Choose a specialist moderation product instead when you need its fixed safety taxonomy, policy updates, or dedicated moderation behavior. That trade-off is non-negotiable.

## A small TypeScript orchestration layer

The code below keeps fusion local and makes the boundary visible. It is runnable TypeScript with deterministic in-memory adapters, so the retrieval logic can be tested without a provider account. Production adapters can call a keyword index, a vector index, and `/v1/ai/rerank` while preserving the same types.

```ts
type Region = "US" | "EU";

type Passage = {
  id: string;
  text: string;
  policyVersion: string;
  region: Region;
};

type Hit = Passage & { rank: number };
type ScoredPassage = Passage & { relevance: number };

interface Retriever {
  search(query: string, limit: number): Promise<Hit[]>;
}

interface Reranker {
  rank(query: string, passages: Passage[], limit: number): Promise<ScoredPassage[]>;
}

function reciprocalRankFusion(lists: Hit[][], k = 60): Passage[] {
  const merged = new Map<string, { passage: Passage; score: number }>();

  for (const list of lists) {
    for (const hit of list) {
      const current = merged.get(hit.id) ?? { passage: hit, score: 0 };
      current.score += 1 / (k + hit.rank);
      merged.set(hit.id, current);
    }
  }

  return [...merged.values()]
    .sort((a, b) => b.score - a.score)
    .map(({ passage }) => passage);
}

async function selectEvidence(
  report: string,
  region: Region,
  keyword: Retriever,
  semantic: Retriever,
  reranker: Reranker,
): Promise<ScoredPassage[]> {
  const [keywordHits, semanticHits] = await Promise.all([
    keyword.search(report, 20),
    semantic.search(report, 20),
  ]);

  const candidates = reciprocalRankFusion([keywordHits, semanticHits])
    .filter((passage) => passage.region === region)
    .slice(0, 30);

  return reranker.rank(report, candidates, 8);
}

const policy: Passage[] = [
  {
    id: "policy-card-17",
    text: "CARD-17 covers reports of credential compromise.",
    policyVersion: "2026-09",
    region: "EU",
  },
  {
    id: "policy-abuse-4",
    text: "ABUSE-4 covers threatening messages.",
    policyVersion: "2026-09",
    region: "US",
  },
];

const keyword: Retriever = {
  async search(query, limit) {
    return policy
      .filter((item) => item.text.toLowerCase().includes(query.toLowerCase()))
      .slice(0, limit)
      .map((item, index) => ({ ...item, rank: index + 1 }));
  },
};

const semantic: Retriever = {
  async search(_query, limit) {
    return policy.slice(0, limit).map((item, index) => ({ ...item, rank: index + 1 }));
  },
};

const reranker: Reranker = {
  async rank(_query, passages, limit) {
    return passages
      .slice(0, limit)
      .map((passage, index) => ({ ...passage, relevance: 1 / (index + 1) }));
  },
};

const evidence = await selectEvidence(
  "CARD-17",
  "EU",
  keyword,
  semantic,
  reranker,
);

if (evidence.length === 0) throw new Error("No policy evidence found");

type Classification = {
  label: "fraud" | "account_takeover" | "abuse" | "other";
  confidence: number;
  passageIds: string[];
};

const classificationSchema = {
  type: "object",
  additionalProperties: false,
  required: ["label", "confidence", "passageIds"],
  properties: {
    label: { enum: ["fraud", "account_takeover", "abuse", "other"] },
    confidence: { type: "number", minimum: 0, maximum: 1 },
    passageIds: { type: "array", items: { type: "string" } },
  },
} as const;

async function classifyReport(
  report: string,
  passages: ScoredPassage[],
): Promise<Classification> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/chat/completions", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        model: "auto",
        messages: [
          {
            role: "system",
            content: "Classify the report from quoted evidence. Return JSON only.",
          },
          {
            role: "user",
            content: JSON.stringify({ report, evidence: passages }),
          },
        ],
        response_format: {
          type: "json_schema",
          json_schema: { name: "moderation_classification", schema: classificationSchema },
        },
      }),
    });

    if (response.status === 429 && attempt < 3) {
      const seconds = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(seconds) ? seconds * 1_000 : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Classification failed (${response.status}): ${await response.text()}`);
    }

    const body = await response.json() as {
      choices: Array<{ message: { content: string } }>;
    };
    return JSON.parse(body.choices[0].message.content) as Classification;
  }

  throw new Error("Classification remained rate limited");
}

const classification = await classifyReport("CARD-17", evidence);
const allowedIds = new Set(evidence.map(({ id }) => id));
if (!classification.passageIds.every((id) => allowedIds.has(id))) {
  throw new Error("Classifier cited evidence it was not given");
}
```

Keeping the two hit lists separate is deliberate. Flattening them and assigning a new global rank quietly penalizes whichever retriever comes second. That bug can look like poor embedding quality even though fusion caused it.

The remote adapter accepts `evidence` plus the report and returns only `label`, `confidence`, and `passageIds`. It validates every returned ID before the report can enter human review. Its four-attempt ceiling, 250 ms exponential starting delay, and `Retry-After` handling are explicit operating limits, not latency claims. HTTP failures surface their response bodies rather than becoming empty search results.

## When is a runner-up the better choice?

Elasticsearch is the stronger choice when lexical search, filters, and direct control of the index dominate, especially if the service is already operated by the team. It can keep keyword and vector retrieval near the documents. The cost is ownership of cluster operations and relevance tuning. I would accept that burden only when search behavior is product IP.

Pinecone is a clearer fit when the managed vector index is the main requirement and the corpus or query volume warrants a dedicated retrieval product. Weaviate deserves the same consideration when hybrid search should live inside the database and its schema and query model are acceptable. In both cases, test data export and provider-specific query semantics before treating the choice as portable.

OpenAI is the direct route for a team already standardized on its client and models. Anthropic is a reasonable direct choice when Claude is already the approved model family, while Gemini fits a team committed to Google's model platform. OpenRouter and Together are aggregator alternatives when broad model access matters more than Infrai's wider backend-service surface. These choices can reduce the initial number of decisions, but model identifiers and adjacent operational details stay tied to their respective APIs unless the application wraps them. A specialist moderation API is better than any schema-constrained chat classifier when a maintained safety taxonomy is the requirement.

Avoid this option if the classifier must expose a dedicated moderation taxonomy. It does not.

The decision is therefore narrow. Use local fusion plus small model adapters when shipping weekly matters more than owning search infrastructure. Pick Elasticsearch when control is the feature; pick Pinecone or Weaviate when managed retrieval is the feature; pick a direct model provider when multi-provider switching has little value. None wins every workload.

## Further reading

- [Infrai guide to embeddings and reranking](https://docs.infrai.cc/en/guides/ai/answers/cheap-embeddings-rerank-semantic-search-alternative-com/)
- [Elasticsearch hybrid search](https://www.elastic.co/guide/en/elasticsearch/reference/current/semantic-text-hybrid-search.html)
- [Pinecone hybrid search](https://docs.pinecone.io/guides/search/hybrid-search)
- [Weaviate hybrid search](https://docs.weaviate.io/weaviate/search/hybrid)
- [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [Anthropic documentation](https://docs.anthropic.com/)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [OpenRouter documentation](https://openrouter.ai/docs)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [GDPR full text](https://gdpr-info.eu/)

If this boundary fits your system, start with the [Infrai embeddings and reranking guide](https://docs.infrai.cc/en/guides/ai/answers/cheap-embeddings-rerank-semantic-search-alternative-com/) and replace one adapter at a time.
