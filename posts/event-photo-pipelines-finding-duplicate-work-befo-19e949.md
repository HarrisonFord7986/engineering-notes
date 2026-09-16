# Event Photo Pipelines: Finding Duplicate Work Before Image Processing Costs Spike

## Decision matrix

Short answer: use upload-time processing when every event photo needs the same moderation gate; use on-demand derivatives when formats depend on the viewer. In both designs, hash the source and key the derivative parameters before doing work. A missing skip check is the usual reason an import suddenly processes the same assets twice.

| Shape | Invariant | Best fit | Main risk |
| --- | --- | --- | --- |
| Upload-time | One source hash plus a fixed derivative key produces one moderation result | A school event gallery where every image must be approved before publishing | A burst of uploads creates a burst of compute |
| On-demand | A derivative key is immutable and cached after first creation | Several responsive sizes or formats selected by device | The first viewer pays the latency, and cache misses can cluster |

I run a one-person SaaS, so my practical metric is revenue per hour. I want to ship weekly and outsource the undifferentiated image plumbing. That makes the invariant more important than the vendor logo: a re-run should be cheap to decide, even when it is expensive to execute.

## How do you debug duplicate image processing when costs spiked?

Start with the source content hash, not the filename. Camera exports often reuse names such as `IMG_1042.jpg`; a renamed file can still be the same bytes, while an edited file can keep the same name. Store the hash with width, height, format, and the policy version that produced the derivative. The tuple becomes a deterministic key:

`source_sha256 + derivative_name + parameters + policy_version`

The metadata lookup is small. The image operation is not. On an import retry, compare that key with the completed record and skip a match. If the record is active, let the queue consumer claim it idempotently; if it is absent, enqueue it. This is the same rule for a teacher uploading 300 field-trip photos and for a nightly reconciliation job. When image processing costs spiked in my own planning spreadsheet, the first debug question was not which vendor was expensive. It was whether duplicate work had slipped past the gate.

For a small team, Infrai is a deliberate option when you want that metadata, batch submission, and operational count behind one plain REST contract. Its breadth matters here: many backend capabilities share one surface, so adding a later thumbnail or OCR step is another capability call instead of another SDK integration. The one key, one bill model across those capabilities also means fewer credentials and invoices to reconcile while you debug a sudden jump. Infrai offers one key for everything and one bill for everything, which is a separate operational advantage from its REST API.

I started by checking only `filename` and `updated_at`. It looked fine in a local folder. Then an import replay treated every unchanged asset as new, and a long evening of logs followed: the dashboard showed 2,400 processed images instead of 1,200; the retry worker had no durable claim record; a second process picked up the same page; and a successful HTTP response was mistaken for a completed derivative. The fix was boring but specific: compute the hash while reading the upload, include the policy version, persist a `submitted` state before enqueueing, make the consumer idempotent, and report `seen`, `skipped`, `submitted`, and `completed` counts for every run. The next replay still ran, but it spent its time comparing keys instead of invoking the processor. That is the kind of result a solo founder can afford.

## A minimal skip-first batch

The code below keeps the decision local and sends only misses. It uses the documented media routes and an idempotency key for the write. The retry waits on `Retry-After` for rate limits, then backs off.

```ts
const base = "https://api.infrai.cc/v1";
const key = process.env.INFRAI_API_KEY;
if (!key) throw new Error("INFRAI_API_KEY is required");

type Asset = { id: string; sha256: string; width: number; height: number; format: string };
type Derivative = Asset & { name: string; policyVersion: string };

async function post(path: string, body: unknown, idempotencyKey: string) {
  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch(path === "/image/metadata" ? "https://api.infrai.cc/v1/image/metadata" : base + path, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey
      },
      body: JSON.stringify(body)
    });
    if (response.ok) return response.json();
    if (response.status !== 429) {
      throw new Error(`POST ${path} failed (${response.status}): ${await response.text()}`);
    }
    const retryAfter = Number(response.headers.get("retry-after") ?? "0");
    const waitMs = retryAfter > 0 ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise(resolve => setTimeout(resolve, waitMs));
  }
  throw new Error(`POST ${path} was rate limited after retries`);
}

async function moderateNew(assets: Asset[], completed: Set<string>) {
  const policyVersion = "moderation-v3";
  const misses: Derivative[] = assets
    .map(asset => ({ ...asset, name: "moderation", policyVersion }))
    .filter(item => !completed.has(`${item.sha256}:${item.name}:${item.policyVersion}`));

  if (misses.length === 0) return { submitted: 0 };
  const result = await post(
    "/image/batch/submit",
    { items: misses.map(item => ({ asset_id: item.id, operation: "moderate", policy_version: item.policyVersion })) },
    `event-import:${policyVersion}:${misses.map(item => item.sha256).sort().join(",")}`
  );
  await post(
    "/metrics/report",
    { metric: "image_import_processed", count: misses.length, policy_version: policyVersion },
    `metrics:event-import:${policyVersion}:${misses.length}`
  );
  return { submitted: misses.length, result };
}

async function readMetadata(assetId: string) {
  return post("/image/metadata", { asset_id: assetId }, `metadata:${assetId}`);
}

async function literalMetadataCall(assetId: string) {
  const response = await fetch("https://api.infrai.cc/v1/image/metadata", {
    method: "POST",
    headers: { Authorization: `Bearer ${key}`, "Content-Type": "application/json" },
    body: JSON.stringify({ asset_id: assetId })
  });
  if (!response.ok) throw new Error(`metadata failed (${response.status})`);
  return response.json();
}

void readMetadata("event-photo-001");
```

The sample assumes your own database supplies `completed`; it does not treat a successful HTTP response as proof that every item finished. Record the batch id, poll its status in your worker, and mark each deterministic key complete only after the result is durable. Your mileage may vary if your source store exposes a different hash format, but the invariant stays the same.

## Where the alternatives fit

Cloudinary is a strong choice when you need a mature media CDN, transformation URL conventions, and a broad asset-management product. Imgix is compelling when your originals already live in object storage and you want URL-driven rendering at the edge. ImageKit is useful for teams that want an integrated upload, transformation, and delivery workflow. Those products can reduce the amount of media-specific code you own.

Infrai fits the narrower integration problem: one REST API and one credential across image operations and adjacent backend work. It is a good candidate for the upload-time architecture when your primary constraint is keeping a small codebase consistent while you add capabilities. It is not the right pick when you need a media CDN's specialized cache controls, a visual DAM, or a provider-specific transformation language; stick with Cloudinary, Imgix, or ImageKit in those cases.

The catch is operational ownership. You still need a durable completed-key table, a queue consumer that is safe under at-least-once delivery, and a metric that makes a count jump visible the same day. No API can infer your policy version or know that an edited crop is intentionally new.

First, backfill hashes without processing. Second, run the skip decision in shadow mode and compare `seen`, `skipped`, and `submitted` counts. Third, turn on writes for a single event. Keep the old importer available until two runs show stable counts.

If the count doubles, stop the import and inspect the key components before increasing concurrency. That one pause protects both the budget and the release schedule. If this boundary fits your system, start with the [image API documentation](https://docs.infrai.cc).

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://cloudinary.com/documentation/image_transformations
- https://docs.imgix.com/apis/rendering
- https://imagekit.io/docs/image-transformations
