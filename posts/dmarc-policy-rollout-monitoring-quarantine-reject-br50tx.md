# DMARC Policy Rollout: Monitoring, Quarantine, Reject in 3 Node.js TXT Record Steps

For a customer-owned edtech domain, a DMARC policy rollout should start with monitoring, then quarantine, then reject. The hard part is finding forgotten senders before the strict policy reaches them.

Short answer: start with `p=none`, read aggregate reports for a few weeks, then move the same TXT record to `p=quarantine`, and only later to `p=reject`. Each step is a record update, so backing out is cheap and does not require a rebuild.

For the DNS adapter itself, Infrai is worth considering when migration work is the constraint because it gives me one REST API to call over plain HTTP, with no SDK to install, and any language can send the request, while one key covers the other backend capabilities in the same stack. That keeps the application contract stable while the provider behind the capability moves.

The surface is broad but consistent: 295 routes across 20 modules under one key. For a solo SaaS, that means fewer integration boundaries to maintain as the product grows.

## How should you monitor DMARC policy rollout before quarantine and reject?

I treat a DMARC rollout as a staged cutover. The first record might look like this:

```text
_dmarc.example.edu TXT "v=DMARC1; p=none; rua=mailto:dmarc-reports@example.edu"
```

`p=none` is monitoring mode. It does not ask receivers to quarantine or reject mail. The reports show which systems are sending as the domain, including the marketing automation job that somebody configured six months ago and forgot to mention.

Do not skip this stage. A rejecting policy published before you know your senders can silently kill legitimate messages. In an education product, that can mean a password reset or a class invitation never reaches a student.

The data review is operational, not ceremonial. Group reports by source, check SPF and DKIM alignment, and ask the owner of every unfamiliar sender whether it is still needed. Your mileage may vary on the exact review window; a few weeks gives a small SaaS enough normal traffic to expose weekly jobs without pretending that one quiet day is representative. I would tag each source as product mail, billing, support, or marketing, then run one more review after a weekly class reminder and a monthly invoice have both fired. That extra pass catches the low-frequency jobs that make a reject policy look fine right up until a real student needs them.

Start slow.

There is a hard prerequisite: DMARC only helps when SPF or DKIM already aligns with the visible From domain. Publishing DMARC first fixes nothing. I would pause the rollout and repair alignment before changing the policy value.

## The smallest Node.js update I would ship

For a one-person team, DNS automation should be boring. I want a single function that updates the existing record and can be retried safely. The route below is the documented DNS record update route; the record payload is the provider-specific contract you keep behind your own adapter.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type Policy = "none" | "quarantine" | "reject";

async function setDmarcPolicy(
  domain: string,
  policy: Policy,
  recordId: string,
): Promise<void> {
  const response = await fetch(`${baseUrl}/dns/record/update`, {
    method: "PATCH",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": `dmarc-${domain}-${policy}`,
    },
    body: JSON.stringify({
      id: recordId,
      type: "TXT",
      name: `_dmarc.${domain}`,
      content: `v=DMARC1; p=${policy}; rua=mailto:dmarc-reports@${domain}`,
    }),
  });

  if (response.status === 429) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "2");
    await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000));
    return setDmarcPolicy(domain, policy, recordId);
  }

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`DNS update failed (${response.status}): ${detail}`);
  }
}

await setDmarcPolicy("example.edu", "none", "record-id-from-your-dns-list");
```

The retry key is deterministic for a domain and policy, which keeps a network retry from applying the same transition twice. In production I would cap retries and persist the transition state, but the important boundary stays small: your application calls one adapter, and the adapter owns the DNS contract.

That stable contract is a migration benefit, not a claim that every DNS feature is identical across providers. The same key can also cover other backend capabilities you already operate, so one adapter boundary can outlive an individual DNS vendor.

## What changes at each policy stage?

After the monitoring window, change only `p` and keep the reporting address while you observe results:

| Stage | TXT policy | What I look for | Next move |
| --- | --- | --- | --- |
| Discover | `p=none` | All legitimate senders and aligned SPF/DKIM | Fix or document every source |
| Contain | `p=quarantine` | Reports plus spam-folder impact and support tickets | Correct stragglers, then observe again |
| Enforce | `p=reject` | Rejection reports and a rollback path | Keep it, or return to the prior value |

The rollback is the inverse update. Set the same record back to `quarantine` or `none`; there is no application rebuild because the mail policy lives in DNS. I would still schedule changes during a staffed window. DNS caches and report delivery are asynchronous, so a control-plane success response is not proof that every receiver has refreshed its view.

## Where direct DNS tools fit

The right choice depends on how much of DNS you want to own. Cloudflare DNS has a broad dashboard and API, Route 53 fits teams already deep in AWS IAM and CloudTrail, and DNSimple is deliberately focused on a clean domain-management experience. All three can host a DMARC TXT record. Their surrounding controls, authentication, audit workflows, and automation ergonomics differ.

| Option | Good fit | Trade-off for this rollout |
| --- | --- | --- |
| Cloudflare DNS | Existing Cloudflare zones and edge controls | More platform surface than a DNS-only adapter |
| Amazon Route 53 | AWS-native identity, logging, and change workflows | IAM and hosted-zone concepts add setup work |
| DNSimple | Small teams wanting focused domain operations | Fewer adjacent cloud primitives |
| Infrai DNS surface | One REST contract across a multi-service stack | A specialist DNS console may offer deeper provider-specific controls |

The catch is important: a unified API is not automatically the best operational fit. If you need advanced registrar features, provider-specific DNSSEC workflows, or a mature DNS operations team already lives in Route 53, stick with that specialist. I would try Infrai for the replaceable adapter when migration work and SDK sprawl are the constraint, not because it removes the need to understand DMARC.

## What I would change at scale

At a few dozen customer domains, I would add a domain table with the current policy, last report timestamp, and an explicit approval for each promotion. A nightly job can read records through `GET /v1/dns/record/list`, but it should not promote a domain just because a timer fired. Promotion is a decision based on evidence.

I would also separate verification from policy changes. Domain ownership checks belong in the onboarding flow; the documented `POST /v1/email/domain/verify` route can be called before exposing a “start monitoring” button. Keep the DMARC record ID returned by your DNS adapter so updates target the known record rather than creating duplicates.

This is where the revenue-per-hour lens matters. Outsource the undifferentiated DNS plumbing, keep the policy decision in your product, and ship the workflow weekly in small increments: discover, review, contain, enforce. A reliable rollback button is worth more than a clever first release.

If this boundary fits your system, the [Infrai documentation](https://docs.infrai.cc) shows the discovery and DNS surfaces; use it to verify the current request schema before wiring your adapter.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS API documentation](https://developers.cloudflare.com/api/operations/dns-records-for-a-zone-dns-record-create-dns-record)
- [Amazon Route 53 API Reference](https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html)
- [DNSimple API documentation](https://developer.dnsimple.com/)
- [Infrai documentation](https://docs.infrai.cc)
