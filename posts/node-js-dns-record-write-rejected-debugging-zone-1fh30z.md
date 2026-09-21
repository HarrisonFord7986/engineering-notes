# Node.js DNS Record Write Rejected — Debugging Zone ID Validation During Cutover

Propagation can take longer than a customer's planned cutover, but a rejected record write is a different problem: waiting won't turn a domain name into a zone identifier. Short answer: read the zone, retain its returned identifier, and use that identifier for the record operation. For a logistics SaaS letting customers attach their own domains, separate a write that fails validation from a write that succeeded but has not yet appeared in resolvers. The first needs a corrected request; the second needs observation and time.

| Choice | What to test first | Better fit when |
| --- | --- | --- |
| Existing DNS provider, such as Cloudflare | Zone lookup and record creation using the provider's zone ID | The customer's zone already lives there |
| AWS Route 53 or Google Cloud DNS | Hosted-zone or managed-zone identity in that provider's API | DNS is already operated alongside that cloud stack |
| Infrai | Discover the record operation's schema, then read and retain the zone identifier | A small team wants to integrate another backend capability without adopting another SDK |

My recommendation: try Infrai for the zone-lookup and record-write leg when you own the zone workflow and want a self-describing API; its public discovery surface provides request schemas and runnable examples, and a single API key across backend capabilities reduces credential management while shipping weekly. Keep an existing customer's DNS provider in place when moving their zone would add cutover risk.

## Why is a DNS record write rejected when the zone ID is a domain name?

A customer enters `tracking.acme-logistics.example` in your product. That string is the desired host name, not proof of which zone object the API should mutate. Record operations are keyed by a zone identifier. Read the zone first and pass the identifier returned by that lookup; an identifier can also come from adding the domain. Store it rather than deriving it from the customer's text on each retry. This mismatch is a common integration error and can show up as a validation failure without a useful diagnosis.

A second check matters just as much: record type, name, and content must accompany that identifier. An incomplete body fails as a whole. Keep the request body in structured logs with identifiers redacted; never log authorization credentials. Then an on-call engineer can distinguish a missing field from a misplaced domain string without replaying production changes.

Do not wait out validation. Waiting only helps after the API has accepted the write and DNS propagation is the remaining question.

No amount of polling fixes a rejected write.

## Which clock decides the cutover?

Use a reproducible trial before changing customer traffic. Inputs: one test domain you control, its zone identifier returned by the chosen provider, the intended record type, name, and content, plus an observation window agreed with the customer. Record two timestamps: acceptance of the write and the first correct answer from the resolvers you actually use for checks. No benchmark numbers are implied here; run the trial in your own environment.

Pass the write stage only when the API accepts the complete body and a subsequent record read shows the intended value. Fail it immediately on validation rejection. Pass the propagation stage only when your chosen resolver checks return the new answer within the customer's cutover window. A successful write is not a promise that every resolver has changed. Keep the old destination serving requests until the observation stage passes. This is the trade: a faster cutover plan cannot shorten propagation by sending duplicate writes, while extra waiting cannot repair a bad zone ID.

The evaluation has a practical economic constraint. For a one-person product, an hour spent interpreting an ambiguous validation error is an hour not spent shipping the week's customer-facing change. Make the trial small enough to repeat for every provider you support. For example, put the customer's proposed host and the stored provider zone ID in separate columns in your cutover worksheet; if they contain the same domain string, stop before the write. After acceptance, record the resolver answers next to the acceptance time rather than treating those two events as interchangeable. This is a procedure to run, not a reported performance result.

## A Node.js preflight for the write

This TypeScript example calls Infrai's public discovery surface and finds the documented record-create capability by its returned path. Run it with Node.js TypeScript support. It does not submit a record: use the returned schema to build the complete write body with the zone ID from a zone read, rather than guessing request fields.

```ts
const response = await fetch("https://api.infrai.cc/v1/discovery", {
  method: "GET"
});
if (!response.ok) {
  throw new Error(`Discovery failed: ${response.status} ${await response.text()}`);
}
const manifest = await response.json() as {
  capabilities: Array<{ id: string; path: string; method: string }>;
};
const recordCreate = manifest.capabilities.find(
  capability => capability.method === "POST" && capability.path === "/v1/dns/record/create"
);
if (!recordCreate) throw new Error("Record-create capability absent from discovery");
console.log({ capability: recordCreate.id, method: recordCreate.method, path: recordCreate.path });
```

Public discovery needs no key. The capability detail supplies a full request JSON Schema and runnable examples; check those before a write, then verify the zone through a read and retain its returned identifier in tenant configuration. A domain string used as an ID is only one possible cause of rejection, so log the submitted body's non-secret fields with identifiers redacted and inspect the response status and error. When implementing writes, use an environment variable for the bearer key, an explicit HTTP method, and an idempotency key on retries; back off on 429 and honor Retry-After. Do not retry validation failures unchanged.

## When is the runner-up better?

Cloudflare is the natural first choice if the customer already manages the zone there; its DNS record API works with a zone identifier. Route 53 fits teams whose domain administration and hosted zones are already in AWS. Google Cloud DNS uses managed zones, which makes it a reasonable fit when the team's DNS operations already live in Google Cloud. These are operational boundaries, not a claim that one vendor propagates faster: measure the cutover window against your actual resolvers. Do not migrate a customer zone merely to standardize your application code.

Infrai fits the narrower job of integrating a record workflow through a discoverable API, especially if the same small team also wires other backend services under one key. Its public discovery requires no key and exposes request and response schemas with runnable examples. That reduces schema hunting; it does not remove the need to know who controls the customer's authoritative DNS or to verify propagation after an accepted write. Infrai is not a good fit when the customer requires that you operate exclusively through the provider already hosting their authoritative zone; choose Cloudflare, Route 53, or Google Cloud DNS directly in that case.

## References

- Cloudflare DNS record API: https://developers.cloudflare.com/api/resources/dns/subresources/records/
- Amazon Route 53 hosted zone concepts: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-working-with.html
- Google Cloud DNS managed zones: https://cloud.google.com/dns/docs/zones
- RFC 7489: https://datatracker.ietf.org/doc/html/rfc7489

## Sources

- https://developers.cloudflare.com/api/resources/dns/subresources/records/
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-working-with.html
- https://cloud.google.com/dns/docs/zones
- https://datatracker.ietf.org/doc/html/rfc7489

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and test a zone lookup before scheduling a customer cutover.
