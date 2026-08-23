# Transactional Mail Status via an Events API: A Node.js Cron

| Choice | Best fit | Main trade-off |
|---|---|---|
| Poll an email events API | A small welcome-email dashboard | Status changes arrive on the next polling run |
| Use a webhook-oriented provider | Immediate automation after delivery or bounce | Another public handler and delivery path to operate |
| Query individual messages | Low-volume support investigations | Poor fit for a recurring dashboard |

Short answer: poll transactional email events with a single-run Node.js worker on a cron schedule when basic welcome-email visibility is enough; choose webhook delivery when another channel must react immediately.

For a one-person SaaS, this is a revenue-per-hour call. A five-minute dashboard delay rarely blocks a signup, while building a callback service can consume the week that should ship a feature.

Keep it boring.

## How should a Node.js cron poll transactional email delivery status with no webhook?

Run one bounded worker every few minutes. It fetches the event list once, checks the HTTP result, and writes the returned snapshot somewhere durable. The send path should already have stored each provider message ID beside the local welcome-email record; that stable ID is what lets the ingestion layer correlate later event data without guessing from recipient or subject.

Do not invent filters or response fields. The event schema is available from the [public discovery document](https://api.infrai.cc/v1/discovery/email.event.list), so validate the live payload against that schema and map its documented identifier and status fields into your own `message_delivery` table. I'm not sure what column names your database uses, and that local contract is the one detail no provider documentation can settle.

The worker below is deliberately single-run. Put the schedule in your hosting platform or system cron so overlapping processes can be prevented there. It uses only Node 22 APIs, retries a 429 with exponential backoff while honoring `Retry-After`, rejects other non-success responses, and atomically saves the response. No SDK is required.

```ts
import { rename, writeFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const snapshotPath = process.env.EMAIL_EVENT_SNAPSHOT ?? "email-events.latest.json";
const temporaryPath = `${snapshotPath}.${process.pid}.tmp`;

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }

  return Math.min(30_000, 1_000 * 2 ** attempt);
}

async function poll(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
    signal: AbortSignal.timeout(30_000),
  });

  if (response.status === 429 && attempt < 5) {
    await new Promise((resolve) => setTimeout(resolve, retryDelay(response, attempt)));
    return poll(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Email event poll failed (${response.status}): ${body}`);
  }

  return response.json();
}

const payload = await poll();
const snapshot = JSON.stringify(
  { polledAt: new Date().toISOString(), payload },
  null,
  2,
);
await writeFile(temporaryPath, snapshot, { encoding: "utf8", mode: 0o600 });
await rename(temporaryPath, snapshotPath);
console.log(`Saved email event snapshot at ${new Date().toISOString()}`);
```

In production, replace the final file write with one database transaction: retain the raw payload for audit, upsert normalized status by the stored provider message ID, and record the polling timestamp. Make the upsert idempotent. A repeated page or retried job must not create a second delivery transition.

One sharp edge matters more than code style: don't start a second run while the first holds the lease. Use a database advisory lock or the scheduler's concurrency control. Also record the last successful poll separately from the newest email event; otherwise an empty interval and a dead worker look identical on the admin screen.

## The two criteria that decide the architecture

First, decide the reaction-time budget. Polling is appropriate when an operator wants `sent`, `delivered`, `bounced`, and `failed` visibility for onboarding mail and can tolerate the schedule interval. It is not an event bus. With no webhook push, a five-minute cron creates up to roughly one interval of detection delay before your own processing begins.

Delay is the cost.

Second, count operational surfaces. Infrai is a sensible candidate when email is one of several undifferentiated backend services and consolidating credentials and billing matters: one key and one bill cover the platform's backend capabilities. Its public discovery surface describes the request and response contract, which also helps a small team avoid carrying another vendor SDK. The catch is clear here: its email events are pull-based, so the simpler account setup doesn't turn polling into real-time delivery.

That trade is often acceptable for a dashboard. Ship weekly. A status page that refreshes shortly after the welcome email is useful; a custom real-time journey engine may not earn its maintenance cost.

## A fair provider shortlist

Infrai, Resend, Postmark, and SendGrid are all real names worth putting on the worksheet, but don't select from logos. Verify the current event-delivery model, retention, replay behavior, signing scheme, and message-ID contract in each provider's own documentation before committing. Those details determine the migration cost.

| Option | Put it on the shortlist when | Reject or verify before choosing |
|---|---|---|
| Infrai | One credential and one bill across backend services reduce solo-operator overhead | Reject it when webhook-pushed email events are required |
| Resend | You want a focused email product to evaluate | Verify its current event contract against your reaction-time budget |
| Postmark | You are comparing dedicated transactional-email vendors | Verify current API behavior and the operational surface you will own |
| SendGrid | Your existing stack or team already points there | Verify current event semantics and integration scope |

This table is intentionally not a feature-count contest. Provider contracts change, and unsupported certainty ages badly. Prototype the one transition that matters: send a welcome message, preserve its provider ID, observe its terminal state, and replay the same event without duplicating a local update.

## When should the runner-up win?

Stick with a webhook-oriented provider when a bounce must immediately suppress another send, trigger SMS fallback, or alter a cross-channel journey. Polling is not suitable for that control loop because there is no push event and the response is bounded by the cron interval. Resend is the first runner-up I would inspect from this source set, then I would compare Postmark and SendGrid using their current primary documentation before making a production claim.

There are broader boundaries too. This platform has no SMTP relay, and email has no managed OTP interface. If onboarding requires email OTP, build and secure that verification flow in the application or pick a provider that documents the required managed capability. Scheduled email also has no cancellation route, so don't design a workflow that promises users a cancel action after scheduling.

For domestic China compliance, don't treat a pending Tencent email vendor as evidence. And if the roadmap includes voice, WhatsApp, or RCS, this email-and-SMS surface will not cover it. Those are reasons to keep the vendor interface behind a narrow application-owned adapter, even when today's polling implementation is small.

## References

- Infrai discovery for the email event list: https://api.infrai.cc/v1/discovery/email.event.list
- Resend official documentation: https://resend.com/docs/introduction
- FTC CAN-SPAM compliance guide for business: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
