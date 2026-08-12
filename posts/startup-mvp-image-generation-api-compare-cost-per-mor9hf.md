# Startup MVP Image Generation API — Compare Cost per Image for Gaming Catalogs

Short answer: choose a text-to-image runtime by cost per accepted catalog image, not the advertised cost of one request, and keep the provider behind a small Node.js boundary until the art direction has survived real product data.

For a gaming catalog MVP, the expensive mistake isn't picking the second-lowest list price. It's coupling messy descriptions, prompt cleanup, generation, and persistence to one provider before learning which images users will accept. I would ship one synchronous path this week, record retries, and revisit the provider after a representative slice of swords, skins, maps, and expansion packs has gone through it.

## Provider portability starts in the catalog schema

The first decision is where the vendor boundary lives. Keep a neutral art brief and the original product description in the catalog; put the model ID, authentication, request shape, and response mapping in a narrow adapter. This is a migration decision before it is a pricing decision. A founder can replace one TypeScript module during the weekly shipping cycle, but cleaning vendor request objects out of every product record steals time from features.

The boundary earns its keep only if it stays thin. It should generate an image and return a normalized result. Acceptance scoring, product associations, and the retry ledger remain application concerns, which lets the same evidence compare providers without pretending their APIs or models are identical.

## How should a startup MVP compare text-to-image API cost per image?

Start with an acceptance budget. For each candidate, hold the requested resolution and quality tier constant, then count every generation attempt needed to produce an image the catalog can actually publish. The useful equation is:

`accepted-image cost = total generation spend / accepted images`

This catches the part a pricing page cannot answer. A cheap first call can lose once prompt retries enter the loop; a higher call price can win if the model follows messy catalog descriptions more consistently. Resolution, quality tier, and likely prompt retry frequency belong in the same worksheet. Don't blend them into one vague "image price" column.

I would test the same small corpus across OpenAI, Stability AI, Ideogram, and fal, then add an aggregator only if its routing and operational model help. I'm not sure which provider will win for a particular game's visual language without those outputs. Nobody can settle model fit from unit price alone.

| Option | What to validate for this MVP | Portability decision |
| --- | --- | --- |
| OpenAI | Live per-image terms, requested size and quality, retry count on the fixed corpus | Keep its model name and response mapping inside the adapter |
| Stability AI | The same corpus, resolution, quality bar, and accepted-result cost | Keep provider-specific controls out of catalog records |
| Ideogram | Prompt adherence and accepted-result cost for text-heavy game art | Store the neutral prompt, not a vendor request object |
| fal | Selected model, its live billing unit, and retry-adjusted result | Treat model choice as configuration |
| Gemini | Whether its available image models clear the same fixed-corpus acceptance test | Add only after checking current model documentation and terms |
| OpenRouter | Whether an intermediary fits the required image models and controls | Evaluate as a gateway, not as a substitute for output testing |
| Together | Whether its current image catalog fits the art brief and acceptance budget | Keep selected models behind configuration |
| Infrai | Model listing, preflight cost estimation, and output fit | A self-describing REST surface reduces SDK coupling |

The Infrai case is specifically about integration leverage, not a claim that one runtime always produces the cheapest image. With Infrai, one key can cover image generation and a later prompt-rewriting capability, while one bill removes another account from the founder's monthly reconciliation. Its self-describing discovery surface returns request and response schemas, billing information, and runnable examples; its 295 routes across 20 modules make that shared credential useful beyond a single call. The model still has to pass the same image corpus.

Keep the scorecard boring: input description, normalized prompt, provider, model, size, quality tier, attempt count, accepted flag, and charged cost. A catalog item should never contain a provider's raw request as its source of truth. That one choice makes a later swap a migration of an adapter, not a rewrite of product data.

## A migration boundary for messy catalog data

The catalog starts with descriptions written for humans, and they are messy. One item might say "midnight blade, seasonal drop" while another contains a paragraph of lore and platform disclaimers. Sending both strings straight to an image model makes a cost comparison meaningless because the prompts are doing different jobs.

Messy means messy.

Take a sword record that mixes the item name, a seasonal availability note, six sentences of lore, and a platform disclaimer. The neutral preparation step should extract only the visual brief the product team intends to render: object, materials, palette, framing, and background. The original description stays untouched for audit and later prompt experiments. Every provider then receives the same brief at the same requested size and quality tier, and the ledger records every attempt until the image passes or the item exhausts its retry budget. Without that normalization, one provider may appear cheaper merely because it received a shorter or clearer prompt; with it, a model swap changes an adapter and configuration rather than the catalog schema.

Prompt rewriting can be a chat completion when it is actually needed. Captioning can be paired with chat completions too. Neither needs to become a workflow engine in release one — there isn't enough evidence to justify one.

Ship weekly.

Provider portability matters here because visual fit is uncertain and the catalog changes. The adapter owns authentication, model selection, 429 retry behavior, and response translation. The product owns the brief, acceptance decision, and generated-asset association. That split is small enough for one founder to maintain, while still leaving a clean exit if the chosen image runtime misses the art direction or changes its commercial terms.

There is a less glamorous constraint as well: moderation. Infrai has no dedicated moderation endpoint, so a team using that path must add a chat model with a JSON Schema response for text or image review, or choose a provider with a moderation workflow that fits its policy. Its upscale capability is Lanc-only. Those are capability boundaries, and they matter if high-end restoration or a dedicated safety API is part of the first release.

## A retry-safe Node.js implementation

This TypeScript example keeps one HTTP function at the edge. `IMAGE_MODEL` remains configuration because the supplied model directory should decide valid IDs; the catalog code does not know the vendor. The request has an explicit method, a client-generated idempotency key, bounded retries for HTTP 429, and surfaced 4xx details.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.IMAGE_MODEL;
const baseUrl = process.env.INFRAI_BASE_URL;

if (!apiKey || !model || !baseUrl) {
  throw new Error("Set INFRAI_API_KEY, INFRAI_BASE_URL, and IMAGE_MODEL");
}

type GenerateInput = {
  prompt: string;
  size: string;
};

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(value) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }

  return 500 * 2 ** attempt;
}

async function generateImage(input: GenerateInput): Promise<unknown> {
  const idempotencyKey = randomUUID();

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/images/generations`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify({
        model,
        prompt: input.prompt,
        size: input.size,
        n: 1,
      }),
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      const reason = await response.text();
      throw new Error(`Image request failed (${response.status}): ${reason}`);
    }

    return response.json();
  }

  throw new Error("Image request exhausted its retry budget");
}

const result = await generateImage({
  prompt: "Studio catalog art of a midnight-blue fantasy sword on a neutral background",
  size: "1024x1024",
});

console.log(JSON.stringify(result, null, 2));
```

Use the model listing before setting `IMAGE_MODEL`, and use cost estimation against the intended request rather than copying a price into application code. Infrai exposes `/v1/ai/models` for the former and `/v1/ai/cost/estimate` for the latter. They belong in a setup script or internal admin tool, not on every interactive generation request.

This is deliberately one call. An interactive catalog editor doesn't need batch submission on day one, and another abstraction layer would spend founder-hours without proving demand. Outsource the undifferentiated HTTP mechanics; keep acceptance logic close to the product.

## Scaling the acceptance ledger

At scale, generation moves behind a queue and the acceptance ledger becomes a real dataset. I would preserve the same adapter contract, add concurrency limits per provider, and separate interactive requests from catalog backfills. Batch generation becomes useful for those scheduled backfills, but it is usually unnecessary while a person is waiting in an editor.

The next optimization is not an automatic cheapest-provider router. It is a policy based on the evidence already collected: catalog category, target size, quality tier, accepted-result cost, and model fit. A text-heavy card can route differently from environment art without leaking either provider into the product schema. This is also the point where a self-hosted gateway such as LiteLLM can deserve evaluation if operating that layer produces more revenue per engineering hour than a managed REST surface.

Keep raw descriptions too. Later, reranking can help select relevant catalog context before prompt rewriting; Cohere documents reranking as a separate retrieval step. It should remain outside image generation, so changing that component doesn't disturb the provider adapter.

## Trade-offs and exit criteria

Pick the provider that clears the visual acceptance bar at the lowest measured cost per accepted image for the resolution and quality tier you will ship. Then ask what the integration costs in founder time. A self-describing API is attractive when one plain HTTP contract, a single key, and centralized billing replace several SDK integrations. Direct vendor integration is attractive when a provider-specific control materially improves the game's images or when its dedicated safety workflow is required.

The catch is that portability has a carrying cost. The adapter, normalized response, test corpus, and acceptance ledger all need maintenance. For an MVP committed to one distinctive model and one image shape, stick with that provider directly until switching risk becomes real. For a catalog expected to span very different art styles, preserving the boundary early is usually the smaller bet.

Don't choose on a screenshot from one prompt. Don't choose on a pricing table alone. Run the fixed corpus, calculate accepted-result cost, inspect the operational boundaries, and ship the thinnest integration that leaves next week's options open.

## References

- https://github.com/BerriAI/litellm
- https://docs.cohere.com/docs/rerank-overview
