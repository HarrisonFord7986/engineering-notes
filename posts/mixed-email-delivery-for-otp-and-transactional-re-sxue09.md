# Mixed Email Delivery for OTP and Transactional Receipts Without SMTP Coupling

A receipt sender has one job after payment settles: accept the event once and get a useful message on its way. **Short answer: don't make an SMTP relay the shared abstraction for OTP email and transactional receipts; make the application own message intent, deduplication, and provider selection, then put each delivery transport behind a small typed adapter.**

This is mostly an integration-effort decision. SMTP is a valid mail transport, but it is a poor place to encode business meaning such as “this payment receipt was already accepted” or “this login code has expired.” A mixed setup becomes manageable when those rules remain above the transport. For a one-person SaaS shipping weekly, that boundary protects the scarce resource: engineering hours that could produce customer-facing work.

## Why the receipt changed the choice

The tempting plan was one generic `sendMail` function. Give it a recipient, subject, and HTML, then point it at an SMTP relay today and another provider tomorrow. That looks cheap because every message fits the same envelope. The catch appears after a payment processor retries a settlement event, a customer asks support to resend a receipt, or authentication issues a replacement login code. These operations do not have the same identity or retry policy even though all three end as email.

For a receipt, the durable fact is the settled order. The sender should derive a stable operation key from that fact, render the message from stored order data, and record the delivery attempt independently of the payment webhook request. A repeated settlement notification can then resolve to the same send operation instead of creating another logical receipt. A manual resend is different: it should be an explicit new operation tied to the same order, because support intended another delivery. This distinction is application state. An SMTP message ID, an HTTP request ID, or a provider response can help trace an attempt, but none of them decides whether the business operation is new.

OTP raises the stakes. A code belongs to a particular challenge, expires according to authentication policy, and can be superseded by a newer challenge. The mail system should deliver the already-created notification; it should not decide which code can establish a session. Mixing providers doesn't change that ownership. It only changes which adapter carries a given attempt.

Keep that line sharp.

Yahoo's sender guidance is also a useful reminder that transport integration is only part of delivery. It calls for authenticating mail with SPF or DKIM, keeping valid forward and reverse DNS records, using a consistent sending history, and separating bulk or marketing mail from transactional or user mail by IP address or DKIM domain. Those are sender-operating concerns, not arguments for a particular client protocol. A perfectly abstracted SMTP call can still sit behind a poorly operated domain.

## How should a mixed provider setup handle transactional OTP email?

Start with message classes, not vendors. “Order receipt,” “login challenge,” and “marketing update” should be distinct types with explicit policies. The receipt begins only after the application has accepted the settled-payment fact. The login challenge begins only after the authentication component has created challenge state. Marketing belongs on a separate path because its audience, consent, and sender practices differ from one-to-one operational mail.

A useful boundary has three layers:

1. The domain layer creates a stable message intent and decides whether it is new, superseded, or an intentional resend.
2. The dispatch layer selects an eligible transport, applies bounded retry policy, and stores attempt metadata.
3. The adapter translates a typed message into SMTP or an email API and returns a normalized acceptance result.

Acceptance is not delivery.

That sentence prevents a surprisingly expensive category error. The transport may accept a message for processing, but the application still needs a separate model for later delivery observations and support investigations. Likewise, a timeout leaves the caller uncertain about whether the remote side accepted the request. Blindly switching to a second provider at that point can duplicate a receipt. For OTP, it can put two messages in the inbox even if only the newest challenge is valid. The correct response depends on evidence: reconcile an ambiguous attempt when the transport exposes enough status information, or make a deliberate retry under the same application operation rather than treating failover as a fresh business event.

I'm not sure there is one retry interval that fits both login codes and receipts. The sources here do not establish one, and the decision needs actual expiry policy, observed delivery latency, and customer tolerance. What is clear is that retry timing belongs in the per-message policy, not inside a universal SMTP helper.

The mixed-provider rule is therefore modest: route only before an attempt when possible; after an ambiguous attempt, preserve the uncertainty and reconcile it. Don't let every application call choose an arbitrary provider. Centralize that decision so credentials, sender identities, and delivery logs don't spread across the codebase.

## The smallest working TypeScript boundary

This build starts after payment settlement has been validated and stored. It deliberately leaves rendering and provider-specific payloads outside the example. The useful part is the operation boundary: duplicate events reuse one logical send, while an intentional support resend gets a new operation key.

```ts
type Message =
  | {
      kind: "order-receipt";
      operationKey: string;
      to: string;
      orderId: string;
    }
  | {
      kind: "login-code";
      operationKey: string;
      to: string;
      challengeId: string;
      code: string;
    };

type Acceptance = {
  transport: "smtp" | "email-api";
  attemptId: string;
  acceptedAt: string;
};

interface DeliveryAdapter {
  send(message: Message): Promise<Acceptance>;
}

interface SendLedger {
  find(operationKey: string): Promise<Acceptance | undefined>;
  record(operationKey: string, acceptance: Acceptance): Promise<void>;
}

export async function dispatchOnce(
  message: Message,
  adapter: DeliveryAdapter,
  ledger: SendLedger,
): Promise<Acceptance> {
  const existing = await ledger.find(message.operationKey);
  if (existing) return existing;

  const acceptance = await adapter.send(message);
  await ledger.record(message.operationKey, acceptance);
  return acceptance;
}

export function receiptFromSettlement(input: {
  settlementEventId: string;
  orderId: string;
  customerEmail: string;
}): Message {
  return {
    kind: "order-receipt",
    operationKey: `settlement:${input.settlementEventId}:receipt`,
    to: input.customerEmail,
    orderId: input.orderId,
  };
}
```

The in-memory shape of this example is intentionally absent because `find` followed by `record` is not enough for concurrent workers. In production, the ledger needs an atomic claim around `operationKey`, plus states for at least claimed, accepted, and ambiguous. Otherwise two workers can both observe no record and send. That is the concrete constraint that turns a clean interface into a dependable workflow.

There is another deliberate omission: `dispatchOnce` doesn't verify OTP codes. The authentication module should store a digest rather than the raw code, enforce expiry and attempt policy, and atomically consume a successful challenge. The adapter needs the raw code briefly to render the message, but delivery acceptance must never grant a session. Don't log the `Message` union wholesale; its login variant contains a secret.

An SMTP adapter and an API adapter may both satisfy `DeliveryAdapter`, yet they shouldn't be forced to pretend every operational feature is identical. Keep transport-specific status identifiers in a restricted attempt record. Normalize only what the application genuinely uses. Resend's public introduction, for example, documents a REST API and SDKs as its integration surface; that is evidence that an API-shaped adapter is a real option, not evidence that every provider shares one contract.

## What I would change at scale

At low volume, I would keep one primary route per message class, one ledger, and a boring support view that finds attempts by order ID or challenge ID. That is enough to answer the questions that consume founder time: did the application create an intent, which transport accepted it, and was a later send an automatic retry or a human-requested resend? Weekly shipping wins when undifferentiated delivery machinery stays small.

At scale, the first change would be stronger orchestration rather than more providers. Put settlement handling and email dispatch on separate durable work steps. Claim operation keys atomically. Encrypt sensitive message data, redact logs, restrict access to delivery metadata, and measure acceptance and observed outcome by message class. Test duplicate settlement events, concurrent workers, timeouts after remote acceptance, invalid recipient data, and a superseded login challenge. A staging “success” that only checks a mocked adapter proves very little.

Only then would I add provider routing. A second transport is useful when measured incidents, regional requirements, or a necessary capability justify its integration and on-call cost. It is not suitable when the team cannot operate separate credentials, sender authentication, suppression data, status semantics, and reconciliation paths. In that case, stick with one well-understood transport and improve the queue, ledger, and sender practices first.

SMTP remains reasonable when existing infrastructure already operates it well, the language ecosystem has a maintained client, and the needed workflow is plain delivery. An email API can be the simpler adapter when direct HTTP fits the stack and its documented contract covers the required operations. A managed authentication service is the better boundary when owning challenge storage, expiry, abuse controls, recovery, and verification would steal too many feature hours. None of those choices removes the need to authenticate the sending domain and monitor its reputation.

The revenue-per-hour test is blunt: outsource commodity delivery, but keep business identity explicit. For settled-order receipts, that means one durable intent per settlement event, an intentional path for resends, and a transport adapter small enough to replace. For OTP, it means the authentication system remains authoritative even when delivery is mixed. Protocol uniformity is pleasant. Correct ownership ships.

## Sources

- https://resend.com/docs/introduction
- https://senders.yahooinc.com/best-practices/
