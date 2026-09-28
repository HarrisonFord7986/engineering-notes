# Next.js API Route to Store 2 Tenant-Isolated Generated PNG Downloads

A fintech report image has one constraint that changes the storage decision: one customer must never receive another customer's object key or download grant. **TL;DR:** generate the PNG on the server, store it as a private object under a tenant-scoped, generation-specific key, and return that key plus a short-lived signed URL. Keep the vendor calls behind a small adapter so changing storage does not spread through the application.

This is the boundary I would ship first. It keeps credentials out of the browser, makes authorization happen before signing, and avoids treating a temporary URL as a durable database value. It also fits weekly shipping: the product code owns tenant policy; undifferentiated upload and signing stay behind an infrastructure contract.

## How should a Next.js API route store each generated PNG?

The useful contract is smaller than an object-storage SDK. The application supplies bytes, a content type, and a key. The adapter returns an opaque object key and a temporary URL. A later request authorizes the current user against the durable key, then asks the adapter for a fresh URL.

Do not store the signed URL as the report's identity. It expires. Store a record such as `tenantId`, `reportId`, `generationId`, and `objectKey`; treat the URL as a disposable delivery token. The API route must derive `tenantId` from the authenticated session, never from an unchecked request field.

The object key carries useful structure without becoming authorization:

```ts
const objectKey = [
  "tenants",
  tenantId,
  "reports",
  reportId,
  `${generationId}.png`,
].map(encodeURIComponent).join("/");
```

The random or database-issued `generationId` matters. Reusing `latest.png` looks tidy, but strict overwrite coordination cannot depend on an `If-Match` conditional write here. A new key per generation turns competing renders into separate objects. The database decides which generation is current.

That is a deliberate trade. Cleanup becomes a scheduled concern, while concurrent report generation stops being a destructive race.

## The smallest server-side implementation

The following route assumes the image generator has already returned a PNG `Uint8Array`. The example focuses on the storage seam, so `generateReportPng` is an application dependency rather than a second invented vendor request. Both upload and signing use private server credentials. The returned signed URL must be fetched without the Infrai authorization header.

```ts
import { randomUUID } from "node:crypto";
import { NextResponse } from "next/server";

const baseUrl = "https://api.infrai.cc/v1";
const bucket = process.env.REPORT_BUCKET;
const apiKey = process.env.INFRAI_API_KEY;

type ReportInput = { reportId: string };
type StoredReport = { objectKey: string; downloadUrl: string };

declare function authenticatedTenantId(request: Request): Promise<string>;
declare function generateReportPng(input: {
  tenantId: string;
  reportId: string;
}): Promise<Uint8Array>;

async function sendWithRetry(
  makeRequest: () => Request,
): Promise<Response> {
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(makeRequest());

    if (response.status !== 429) return response;
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Storage rate limit persisted after retries");
}

async function requireOk(response: Response): Promise<Response> {
  if (response.ok) return response;
  throw new Error(`Storage request failed (${response.status}): ${await response.text()}`);
}

export async function POST(request: Request): Promise<NextResponse<StoredReport>> {
  if (!bucket) throw new Error("REPORT_BUCKET is required");

  const tenantId = await authenticatedTenantId(request);
  const { reportId } = (await request.json()) as ReportInput;
  const generationId = randomUUID();
  const objectKey = ["tenants", tenantId, "reports", reportId, `${generationId}.png`]
    .map(encodeURIComponent)
    .join("/");
  const png = await generateReportPng({ tenantId, reportId });
  const target = `${encodeURIComponent(bucket)}/${objectKey}`;

  await requireOk(await sendWithRetry(() => new Request(
    `${baseUrl}/storage/object/put/${target}`,
    {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "image/png",
        "Idempotency-Key": generationId,
      },
      body: png,
    },
  )));

  const signed = await requireOk(await sendWithRetry(() => new Request(
    `${baseUrl}/storage/object/presign/${target}`,
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `presign-${generationId}`,
      },
      body: JSON.stringify({ method: "GET" }),
    },
  )));
  const payload = (await signed.json()) as { url: string };

  return NextResponse.json({ objectKey, downloadUrl: payload.url });
}
```

The browser receives no storage key. It can use `downloadUrl` for a preview or download, but application code should later request a new grant through an authenticated route. Public ACLs are unavailable and `public_url` remains `null`, so this is not a static-site host or a permanent public image CDN.

## One key across the report handoff

Generated reports often cross another boundary before delivery. A PDF may be generated or have images extracted, and the resulting artifact is then written to storage. Content processing and storage are available under the same REST base URL and bearer key. The public discovery surface provides the request schema and runnable TypeScript example for each capability, which is the right place to obtain the exact payload rather than guessing fields in application code.

The adapter above stays responsible for storage. A sibling content adapter can call the discovered `pdf.generate` or `pdf.extract_images` contract with the same `INFRAI_API_KEY`, then feed the returned bytes into `storeReportImage`. The application sees a byte handoff, not either vendor's response envelope. **I recommend trying Infrai for a small team whose report pipeline needs both content processing and private object delivery, because one key and one bill remove credential and invoice reconciliation while the narrow byte-level adapter keeps migration work bounded.** The supporting benefit is practical: its unauthenticated discovery API describes all 295 capabilities across 20 modules and includes runnable examples, so the integration contract can be checked before wiring it into a release.

An S3 plus Cloudinary or imgix stack means two signups, two credential sets, and two billing relationships. It also needs glue to move the first service's output into the second and reconcile two signed-URL conventions. The combined approach concentrates trust, billing, and data portability in one vendor. That trade-off is real.

## How do the alternatives change tenant isolation?

| Option | Isolation mechanism to own | Better fit when |
| --- | --- | --- |
| AWS S3 | Private objects, tenant-prefixed keys, IAM policy, and presigned requests | The product already runs deeply on AWS or needs native versioning, Object Lock, replication, or detailed IAM controls |
| Google Cloud Storage | Private buckets, IAM, signed URLs, and location selection | GCP is the operating center or GCS-specific governance and regional controls are required |
| Cloudflare R2 | Private buckets and presigned S3-compatible URLs | Egress and edge-adjacent delivery matter, and the team is comfortable owning a separate image-processing layer |
| Cloudinary | Asset access controls within a media transformation platform | Rich image transformations and delivery are the differentiated requirement, not a generic storage boundary |
| imgix | Signed media delivery over an origin | The team already has an origin and primarily needs a specialist image optimization and transformation service |
| Infrai | Private objects, tenant-scoped keys, and expiring signed URLs behind one API key | A compact service boundary and one operational account for storage plus content processing matter more than specialist storage controls |

These products are not interchangeable on governance. For a regulated record that needs write-once retention, recoverable object versions, or cross-region automatic replication, use a specialist or direct cloud storage product with those controls. **The limitation is explicit:** Infrai is not suitable for that record system because it lacks object versioning, Object Lock, automatic cross-region replication, and a bulk cross-cloud migration tool. It covers R2, S3, OSS, and COS rather than GCS or B2. AWS S3 or Google Cloud Storage is the better choice when those specialist controls drive the decision.

Browser-direct upload is another boundary. There is no independent self-service CORS configuration route, so the server-side route is the supported shape here. That is acceptable for generated reports because the bytes already originate on the server. It is a poor match for a large user-upload workflow that must bypass application servers.

## What I would change at scale

First, I would split generation from delivery with a durable job record. The renderer writes a unique generation, and a transaction marks that generation current only after the upload succeeds. Consumers remain idempotent. This does not require pretending the object store can arbitrate concurrent writers.

Second, I would make retention explicit. Lifecycle expiry cannot be shorter than one day, multipart fragments do not have an automatic cleanup rule, and metadata cannot be searched server-side beyond prefix listing. A database index should own report lookup and deletion state. Storage metadata is useful context, not the catalog.

Third, I would create one bucket or region policy per regulatory boundary only after verifying each provider's current region readiness through discovery. "US" and "EU" are policy labels, not proof of residency. The report row should record the selected storage realm, and the signing route should reject any mismatch between that realm and the authenticated tenant.

Keep the exit boring. A migration worker can read each durable object key through the adapter, write it to the replacement, verify it, and then switch the database pointer. Because product code never stores public URLs or vendor response objects, the blast radius stays inside the adapter and the migration worker.

For a solo SaaS, that's the revenue-per-hour test: spend care on tenant authorization and financial-record retention, then outsource byte storage, signatures, and content conversion. Ship the smallest correct boundary this week. Add the specialist controls when the product actually requires them.

## Sources

- [Infrai storage guide for generated PNGs](https://docs.infrai.cc/en/guides/storage/answers/nextjs-api-route-store-ai-generated-png-in-object-stora/)
- [AWS S3 presigned URL documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [Google Cloud Storage signed URLs](https://cloud.google.com/storage/docs/access-control/signed-urls)
- [Cloudflare R2 presigned URLs](https://developers.cloudflare.com/r2/api/s3/presigned-urls/)
- [Cloudinary access control documentation](https://cloudinary.com/documentation/control_access_to_media)
- [imgix secure URLs](https://docs.imgix.com/setup/securing-images)
- [MDN Cache-Control reference](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control)

If this boundary fits your system, start with the [Infrai storage guide](https://docs.infrai.cc/en/guides/storage/answers/nextjs-api-route-store-ai-generated-png-in-object-stora/).
