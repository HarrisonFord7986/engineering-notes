# Transactional Email Reset Link: Fix Multiple Sends with an Idempotency Key

Short answer: give each password-reset operation one durable idempotency key, commit it with an outbox row, and never turn a timed-out request into a new logical send. A transport that honors that key is the safest default; without transport-side deduplication or a reliable status lookup, true exactly-once submission across a network boundary is impossible.

| Choice | Duplicate control after a timeout | Best fit | Limitation |
| --- | --- | --- | --- |
| Durable operation key accepted by the mail transport | The same key identifies every submission attempt | Security-sensitive mail such as password recovery | Requires explicit transport support and a documented deduplication window |
| Transactional outbox plus delivery lookup | The application reconciles an uncertain attempt before resubmitting | Existing mail systems with searchable submission records | Lookup retention and consistency become part of the design |
| Database outbox only | Stops duplicate workers before the network call | Ordinary notifications where an occasional duplicate is tolerable | Cannot resolve a lost response after remote acceptance |
| Short coalescing window | Merges bursts close together in time | Repeatable marketplace alerts such as several new-order events | Time is not identity; a later retry can still duplicate an earlier accepted send |

For a fintech marketplace, use the first row for a seller's password-reset email and keep the outbox as the local source of truth. A coalescing window is the runner-up for lower-risk new-order notifications, where grouping several alerts can be a product choice rather than a security decision.

## How can a password reset email retry after timeout avoid duplicate sends?

A timeout says the outcome is unknown. It does not say the send failed. The remote mail system may have accepted the message while the response disappeared on the return path. If a worker interprets that silence as failure and creates another reset token, the seller can receive multiple emails with multiple links. If both tokens remain valid, the application has created a security-state problem as well as an inbox problem.

The unit of identity must be the reset operation, not the HTTP attempt, queue delivery, or browser click. Create an opaque operation ID when the application accepts the reset request. In one database transaction, store a hash of the reset token, its expiry, the operation ID, and an outbox record. Put a unique constraint on the operation ID. Every worker attempt then loads the same row and passes the same idempotency key downstream.

Keep the raw token out of logs and deduplication fields. The email needs the raw value in its link, but the durable account record should retain only what is required to verify it. When a new reset operation is intentionally created, invalidate the older operation's token according to the account policy. A retry of the old operation is different: it reuses the old identity and must not mint another token.

This distinction is the whole fix.

Consider a hypothetical trace. Attempt 1 starts at `08:30:00` after the seller follows a reset prompt from the marketplace sign-in screen. The worker commits outbox operation `reset_7f2`, changes it from `ready` to `submitting`, and calls the mail transport. Two seconds later the client stops waiting, but no rejection arrives. At `08:30:12`, the queue makes attempt 2 visible because attempt 1 never acknowledged the job. The database still says `submitting`, not `failed`, so the second worker locks `reset_7f2` and checks the transport using that exact key. If the first call was accepted, the lookup supplies its submission identifier and the worker records `accepted`. If the transport lookup has not caught up, the worker records `uncertain` and releases the lock for a later reconciliation pass. If no lookup exists but the transport honors idempotency keys, the worker may repeat the call with `reset_7f2`; the transport contract, rather than timing luck, suppresses another acceptance. At no point does the worker create `reset_7f3`, generate another link, or infer failure from silence. A response that explicitly rejects the request can move through a documented retry policy. Missing evidence cannot be promoted into evidence of failure.

Exactly once needs careful wording here. An application can enforce one logical reset operation. A cooperating transport can enforce one accepted submission for an idempotency key. Neither condition proves that every downstream mail relay or mailbox will display exactly one copy. DKIM authenticates responsibility for a signing domain and message content; it does not deduplicate messages or settle an ambiguous submission outcome. Treat authentication and duplicate control as separate layers.

## The two criteria are identity and evidence

Identity comes first because retries occur at several layers. A user can click twice. A web process can retry a database transaction. A queue can redeliver after its visibility timeout. A mail client can time out after the remote side has committed the request. One stable operation ID has to survive all four paths, or each layer can honestly believe it is handling a new job.

No new key.

Use a database uniqueness constraint, not an in-memory flag. Two workers can start on different machines, and a process can die after sending but before updating local state. A useful local state machine distinguishes `ready`, `submitting`, `accepted`, `uncertain`, and `rejected`. `uncertain` is deliberate. It prevents a second logical send while an operator or reconciliation job looks for evidence of remote acceptance.

Evidence is the second criterion. Before choosing a mail transport, verify two claims in its current technical documentation: whether it accepts a caller-supplied idempotency key, and whether a submission can be retrieved by that key after a client timeout. Record the retention window, lookup consistency, and key scope. I'm not sure those guarantees remain identical across service plans or over time; a contract test against the selected transport, plus a dated architecture note, resolves that uncertainty better than an assumption in application code.

The operational metric should reflect the state machine. Count logical reset operations, submission attempts, uncertain outcomes, reconciliations, and accepted submissions separately. Alert when an operation has more than one accepted submission identifier or remains uncertain beyond the reconciliation deadline. Don't use mailbox opens as proof of acceptance: image blocking, privacy features, and forwarding make engagement a poor control signal.

This is undifferentiated plumbing, so keep it small. One table, one worker, one adapter contract, and a few sharp metrics are enough. The revenue-per-hour lens favors a boring invariant that can ship this week over a large messaging framework that still cannot explain what happened after a timeout.

## A TypeScript boundary that preserves the operation key

The adapter below makes the important behavior visible. It doesn't pretend that a local process can infer a remote result. The same `operationId` is used for every call, and an uncertain result leaves the record in a state that requires reconciliation.

```ts
type ResetEmail = {
  operationId: string;
  recipient: string;
  resetUrl: string;
};

type Submission =
  | { kind: "accepted"; submissionId: string }
  | { kind: "rejected"; reason: string }
  | { kind: "uncertain" };

interface MailTransport {
  submit(
    message: ResetEmail,
    options: { idempotencyKey: string },
  ): Promise<Submission>;

  findByIdempotencyKey(key: string): Promise<
    | { kind: "accepted"; submissionId: string }
    | { kind: "not-found" }
  >;
}

interface ResetOutbox {
  lock(operationId: string): Promise<{
    message: ResetEmail;
    state: "ready" | "submitting" | "accepted" | "uncertain" | "rejected";
  }>;

  markSubmitting(operationId: string): Promise<void>;
  markAccepted(operationId: string, submissionId: string): Promise<void>;
  markUncertain(operationId: string): Promise<void>;
  markRejected(operationId: string, reason: string): Promise<void>;
}

async function submitResetEmail(
  operationId: string,
  outbox: ResetOutbox,
  transport: MailTransport,
): Promise<void> {
  const row = await outbox.lock(operationId);

  if (row.state === "accepted" || row.state === "rejected") return;

  if (row.state === "uncertain" || row.state === "submitting") {
    const prior = await transport.findByIdempotencyKey(operationId);
    if (prior.kind === "accepted") {
      await outbox.markAccepted(operationId, prior.submissionId);
      return;
    }
  }

  await outbox.markSubmitting(operationId);
  const result = await transport.submit(row.message, {
    idempotencyKey: operationId,
  });

  if (result.kind === "accepted") {
    await outbox.markAccepted(operationId, result.submissionId);
  } else if (result.kind === "rejected") {
    await outbox.markRejected(operationId, result.reason);
  } else {
    await outbox.markUncertain(operationId);
  }
}
```

The database implementation must hold an appropriate row lock or use a compare-and-set update around the transition. The transport contract must also define what `not-found` means. If lookup is eventually consistent, an immediate miss is not permission to create a new operation; wait through the documented reconciliation interval and submit again only with the same idempotency key.

Test the boundary with failure injection. Cover two workers claiming one row, a timeout before any response, a process exit after remote acceptance but before `markAccepted`, delayed lookup visibility, and repeated clicks that refer to the same reset window. The assertion is not merely “one worker ran.” It is that all attempts retain one operation ID, one token generation, and no more than one accepted submission for the key.

Ship that invariant weekly. Template polish can wait.

## When is the runner-up a better choice?

Use an outbox plus lookup when an established transport has no caller-supplied idempotency key but does provide a documented, reliable way to find a prior submission. The catch is that the interval between timeout and reconciliation remains unresolved. During that interval, the worker should wait. This is suitable when reset latency can absorb the lookup window and the team is prepared to operate the `uncertain` queue.

Stick with a database-only outbox or a coalescing window for notifications whose duplicates are annoying rather than security-sensitive. A fintech marketplace might combine several new-order alerts for one seller into a digest keyed by seller and time bucket. That rule is unsuitable for password recovery: two requests in the same minute may represent distinct user intent, while one request retried outside the minute may still be the same operation. A clock cannot answer an identity question.

SMS is also a different channel, not an automatic reliability fallback. Its consent, opt-out, sender identity, and ecosystem requirements need their own design review; CTIA publishes messaging interoperability and compliance principles for that environment. Adding SMS does not repair an ambiguous email state, and sending both channels on every retry can multiply the disturbance.

If the chosen mail transport supports neither idempotent submission nor dependable reconciliation, be honest about the guarantee: the system can suppress concurrent local work, but it cannot ensure exactly-once remote acceptance after a lost response. For password resets, change the transport contract or accept a documented at-least-once risk with one valid token. Don't hide that trade-off behind another queue.

## References

- https://datatracker.ietf.org/doc/html/rfc6376
- https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
