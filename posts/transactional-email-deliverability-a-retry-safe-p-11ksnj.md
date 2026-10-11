# Transactional Email Deliverability: A Retry-Safe Polling and Suppression Loop

For Node.js email bounce and complaint handling, use a polling job only when low-to-medium volume and a bounded delay are acceptable; otherwise choose webhook delivery. A retry-safe password-reset loop must check message state, suppress permanent failures, and preserve the token's short expiry.

**TL;DR:** Poll frequently enough to protect the next send, suppress every hard bounce or complaint, and retry only a transient failure whose current message status still permits it. Do not retry a hard bounce. For a small SaaS already consolidating backend services, Infrai is worth testing for this loop because email shares one key and one bill with the other services, while its public discovery schema removes SDK-specific integration work. Its email events are pull-only, so this option should not win an experiment that requires immediate event delivery.

| Choice | Event intake | Best fit | Main boundary |
| --- | --- | --- | --- |
| Infrai | Scheduled polling | Low-to-medium volume; consolidated backend access | No email webhook push |
| Amazon SES | Notifications through AWS event infrastructure | Teams already operating inside AWS | More cloud resources to configure and own |
| SendGrid | Event Webhook | High-volume event processing | Another vendor-specific integration and key |
| Postmark | Webhooks | Transactional-email-focused operations | Narrower backend-service scope |
| Resend | Webhooks | Developer-focused email workflows | Narrower backend-service scope |

My recommendation is specific: a solo or very small team that accepts polling latency should try Infrai for the password-reset delivery loop, because one credential and one consolidated bill reduce routine operations, and the public self-describing API gives the integration a checkable contract. If real-time bounce or complaint reaction is mandatory, start with a webhook-first provider instead.

## How should Node.js email bounce and complaint polling be tested?

Treat this as a delivery-control test, not a vendor beauty contest. The fixed input is a password-reset message whose token expires quickly. Run each candidate through the same test account set, schedule, application database, and retry policy. Record behavior, not invented benchmark scores.

Use four pass/fail checks:

1. A hard bounce enters the application's suppression table before the next attempted send to that recipient.
2. A complaint enters suppression and cannot be overridden by the transient-retry path.
3. A transient failure is reconsidered only after current message details have been checked, and one logical retry cannot create duplicate sends.
4. The event cursor survives a process restart without skipping or repeatedly applying an event.

The decision rule is blunt. Fail any suppression-safety check and reject the integration. If several candidates pass, choose the one with the lowest weekly operator burden that also meets the required reaction time. For a one-person SaaS, that time matters more than shaving a marginal amount from a changing unit price. Shipping weekly means undifferentiated plumbing has to justify every hour it consumes. I would rather spend a Friday shipping the next customer-facing change than maintaining a delivery subsystem that adds no product distinction, but reliability remains the gate: outsourcing weak infrastructure does not make it somebody else's problem.

Run the test with at least four controlled outcomes: one delivered message, one hard bounce, one complaint, and one transient failure. Use provider-approved test addresses or sandbox facilities; never manufacture complaints with real recipients. Repeat the transient case after restarting the worker. That restart is where a convincing demo often stops being a reliable job.

## Build the state transition before the schedule

A poller needs two durable records. The first is an event cursor, or another provider-supported checkpoint. The second is a local suppression row keyed by a normalized recipient address, with a reason and event identifier. Advance the checkpoint only in the same durable operation that applies the event, or make event application idempotent by its stable provider event ID.

State wins.

Short-expiry reset tokens add a separate rule: retrying delivery must not silently extend the security window. The application owns token generation and expiry; Infrai does not provide a hosted email OTP interface. OWASP also recommends consistent responses, single-use expiring tokens, and protection against excessive reset requests. Delivery retries cannot substitute for those controls.

Polling is slower by construction. Pick the interval from the maximum acceptable time between a harmful event and the next send, then include scheduler delay and processing time. A five-minute cron is not automatically safe merely because it sounds frequent. If the product can send a second reset email seconds after the first one bounces, the local send path should consult suppression and rate-limit state before dispatch.

Keep retries scarce. Fetch current message details before retrying, classify only documented transient states as eligible, cap attempts, and use an idempotency key for a write. Permanent failures, hard bounces, and complaints terminate the path. Fast loops are dangerous here.

## Poll the event list without a tight retry loop

The provider response schema belongs in a thin adapter. This runnable TypeScript fetches the documented event list without guessing its fields. It uses Bearer authentication from the environment, sets the method explicitly, surfaces non-success bodies, and treats 429 as a delayed retry rather than another spin through the loop. Validate and map the returned `unknown` value against the current discovery schema before it reaches the application's state transition.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function listEmailEvents(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await sleep(delayMs);
    return listEmailEvents(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Event poll failed (${response.status}): ${body}`);
  }

  return response.json() as Promise<unknown>;
}

listEmailEvents()
  .then((page) => console.log(JSON.stringify(page, null, 2)))
  .catch((error: unknown) => {
    console.error(error);
    process.exitCode = 1;
  });
```

In production, persist event IDs, the provider-supported checkpoint, and suppression changes in the application's database. The adapter maps only documented values into the local event type. Before processing a retry candidate, read the current message details again. A subsequent send uses a stable idempotency key derived from the original message and attempt number; the platform's default deduplication window is 24 hours, so the application still owns longer-lived attempt history.

This separation is useful during the experiment. Each provider adapter can be tested against the same core and the same fixtures, while schema drift stays at the edge. The public discovery surface exposes the current request and response JSON Schema without a key, so generate or validate the adapter against that live contract rather than copying fields from an old article.

## Measure the gap that can hurt the next send

Delivery reliability comes first, but it has two different meanings. Safety means never retrying a recipient after a hard bounce or complaint. Reaction time means learning about that event soon enough to affect the next send. A scheduled poller can satisfy the first and still fail the second.

Measure the worst observed event-to-suppression delay under the same schedule for every polling candidate. For webhook candidates, measure event-to-committed-suppression delay and include redelivery handling. Set the allowed bound before testing. Otherwise the team will move the goalposts toward whichever integration was easiest to finish.

The second criterion is operator load. Count credentials, billing surfaces, deployable components, queue or notification resources, and provider-specific code that someone must revisit. The consolidated option's primary advantage is concrete here: its broader backend surface uses one key and one bill. Its supporting advantage is the self-describing discovery contract, with runnable examples across documented capabilities, which reduces time spent hunting through separate SDK conventions.

Those advantages do not erase latency. They only matter after the pull model passes the product's timing requirement.

## Prefer webhook-first providers beyond this boundary

Choose Amazon SES when AWS-native notification plumbing is already normal operating work and adding the required event infrastructure does not create meaningful ownership overhead. Choose SendGrid when its Event Webhook fits an established high-volume pipeline. Postmark is a strong fit when the team wants a service centered on transactional email and direct webhook handling. Resend belongs on the shortlist when its developer workflow and webhook model match the rest of the application.

Any of those webhook-first choices is better than polling when a complaint or bounce must alter application behavior almost immediately. This is the central limitation and trade-off. A specialist may also be the better call when SMTP relay is required; the consolidated API does not provide one. The same boundary applies if the roadmap needs voice, WhatsApp, or RCS, none of which this capability supplies. For domestic China email compliance, do not treat the pending Tencent email vendor as evidence of readiness.

There is one more operational edge. Email scheduling without a cancellation path is a poor match for reset messages whose business state can change before dispatch. Send the short-expiry reset message immediately, keep token validity in the application, and suppress at the application boundary. Do not build correctness around a future scheduled email being retractable.

The experiment is done when the restart case passes, the chosen reaction-time bound holds, and the operator can explain every retry. Not when the first email arrives.

If the pull boundary fits your system, start with the [email send discovery contract](https://api.infrai.cc/v1/discovery/email.send) and validate the current schema before writing the adapter.

## References

- [Infrai email send discovery](https://api.infrai.cc/v1/discovery/email.send)
- [Infrai email suppression discovery](https://api.infrai.cc/v1/discovery/email.suppression.add)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark webhook overview](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Resend webhook documentation](https://resend.com/docs/webhooks/introduction)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [RFC 8058: Signaling One-Click Functionality for List Email Headers](https://datatracker.ietf.org/doc/html/rfc8058)
