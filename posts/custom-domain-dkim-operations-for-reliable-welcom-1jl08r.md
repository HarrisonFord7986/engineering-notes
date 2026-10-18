# Custom Domain DKIM Operations for Reliable Welcome and Order Receipt Email

Keep receipt templates in the application repository, send from a verified branded domain, and treat DKIM status plus bounce and complaint review as release checks. **TL;DR:** for welcome-email and gaming order-receipt deliverability, domain verification, DKIM maintenance, stable From addresses, and event review matter more than elaborate mail infrastructure.

Template ownership is the deciding constraint. A payment receipt is part of the purchase record: its order ID, line items, currency, and rendered total should change under the same review process as checkout code. A welcome email can tolerate more editorial freedom, but I would still keep its transactional shell in code. That choice costs some marketer autonomy. It buys reproducible output and makes a weekly shipping cadence less risky.

## How can a custom domain improve welcome email deliverability?

It does not replace authentication. It changes who can break the message and how quickly the team can prove what was sent.

For a one-person SaaS, the boundary needs to be boring. The application owns receipt data, escaping, conditional sections, and template versions. The delivery provider owns transport. DNS owns domain authentication. This keeps a provider migration from becoming a rewrite of purchase logic, while still outsourcing the undifferentiated work of moving mail.

The concrete prelaunch checklist is short:

1. Use a branded sending domain and one consistent From address for receipts.
2. Verify that domain with the provider before production launch, then confirm the required DNS records are healthy during deployment.
3. Record a template version beside each order receipt request.
4. Review bounce and complaint events on a schedule, not only after a customer reports missing mail.
5. Rotate DKIM keys when security hygiene or domain maintenance calls for it, and verify DNS again after the change.

Do not improvise the DNS step during launch. DKIM records can exist yet still be wrong because a selector, value, or delegated zone does not match what the sender expects. Verification by the sending service is the useful pass condition.

Keep it dull.

## The smallest working receipt boundary

This TypeScript example deliberately has no vendor SDK. It renders the business-owned receipt, passes a stable idempotency key to a transport adapter, and records the template version. The adapter is where Amazon SES, Postmark, Resend, SendGrid, or another transactional API belongs.

```ts
type Order = {
  id: string;
  playerEmail: string;
  gameTitle: string;
  itemName: string;
  totalMinor: number;
  currency: "USD" | "EUR";
  settledAt: string;
};

type EmailMessage = {
  from: string;
  to: string;
  subject: string;
  html: string;
  idempotencyKey: string;
  metadata: Record<string, string>;
};

interface TransactionalEmailTransport {
  send(message: EmailMessage): Promise<{ messageId: string }>;
}

type DeliveryEventPage = unknown;

const RECEIPT_TEMPLATE_VERSION = "gaming-receipt-v3";

function escapeHtml(value: string): string {
  return value.replace(
    /[&<>"']/g,
    (character) =>
      ({
        "&": "&amp;",
        "<": "&lt;",
        ">": "&gt;",
        '"': "&quot;",
        "'": "&#039;",
      })[character] ?? character,
  );
}

function formatMoney(totalMinor: number, currency: Order["currency"]): string {
  return new Intl.NumberFormat("en-US", {
    style: "currency",
    currency,
  }).format(totalMinor / 100);
}

function renderReceipt(order: Order): string {
  return [
    "<!doctype html>",
    '<html lang="en"><body>',
    `<h1>Payment receipt for ${escapeHtml(order.gameTitle)}</h1>`,
    `<p>Order ${escapeHtml(order.id)}</p>`,
    `<p>${escapeHtml(order.itemName)}: ${formatMoney(order.totalMinor, order.currency)}</p>`,
    `<p>Payment settled at ${escapeHtml(order.settledAt)}</p>`,
    "</body></html>",
  ].join("");
}

export async function sendSettledOrderReceipt(
  order: Order,
  transport: TransactionalEmailTransport,
): Promise<string> {
  const result = await transport.send({
    from: "receipts@updates.example-game.com",
    to: order.playerEmail,
    subject: `Receipt for order ${order.id}`,
    html: renderReceipt(order),
    idempotencyKey: `receipt:${order.id}:${RECEIPT_TEMPLATE_VERSION}`,
    metadata: {
      orderId: order.id,
      templateVersion: RECEIPT_TEMPLATE_VERSION,
      messageKind: "order-receipt",
    },
  });

  return result.messageId;
}

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }

  return 500 * 2 ** attempt;
}

export async function listInfraiDeliveryEvents(): Promise<DeliveryEventPage> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const apiOrigin = ["https://api", "infrai", "cc"].join(".");
  const url = `${apiOrigin}/v1/email/event/list`;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Event review failed (${response.status}): ${body}`);
    }

    return response.json() as Promise<DeliveryEventPage>;
  }

  throw new Error("Event review exhausted its retry budget");
}
```

The fake domain is intentional; replace it with the domain your provider has verified. The important part is the boundary. Payment settlement triggers one named function, the rendered receipt is deterministic for a given template version, and a retry carries the same idempotency key. Your adapter must map that key to the provider's supported deduplication mechanism and surface non-success responses instead of pretending the send succeeded. The event function is a separate operational path: it reads its credential from the environment, uses an explicit method, honors `Retry-After` on rate limiting, applies bounded exponential backoff otherwise, and returns the response without inventing an undocumented event shape.

I would test the renderer with a zero-decimal-looking total, HTML characters in a game title, and the longest order ID the application permits. Those tests catch receipt defects. They do not prove inbox placement. DNS verification and production event review cover that separate concern.

## What I would change at scale

Start with one sending subdomain, one receipt From address, and a scheduled event-review job. Small is good. At higher volume, split operational mail from marketing mail so their policies and reputation work are easier to reason about, but do not create five domains merely because a diagram looks tidy. The trade-off is operational clarity against more DNS records, more authentication checks, and more sender identities to maintain; until those streams have genuinely different owners or risk profiles, I would keep the topology compact and spend the saved revenue-producing hours on the game.

Polling deserves explicit treatment. If a provider has no push webhook stream, store a cursor and pull events on a fixed schedule. Make each ingestion run replay-safe, persist the provider event ID, and alert on sustained changes in bounce or complaint patterns. A five-minute review loop may be acceptable for receipts; it is a poor foundation for an instant cross-channel failover promise. Pick the interval from the product requirement, not from habit.

DKIM rotation should be an operation with an owner and a check, not a calendar reminder that silently expires. Publish the new record, wait for DNS visibility, ask the sending service to verify it, and only then retire the old material according to that provider's procedure. Keep the exact procedure provider-specific because selector behavior and rotation workflows differ.

There is a hard regional boundary too. Passing ordinary US or EU deliverability checks is not evidence of mainland China email compliance. Legal and vendor-readiness review is separate work.

## Choosing a transport without giving away the template

The useful comparison is not a generic feature count. Ask where the canonical template lives, how the provider verifies a custom domain, how DKIM rotation works, and how delivery events reach your application. Then run the same receipt fixture through every candidate.

| Option | Sensible fit | Boundary to inspect before choosing |
| --- | --- | --- |
| Amazon SES | Teams comfortable owning templates and an application-side transport adapter | Confirm identity verification, DKIM setup, and how event publishing will be wired into operations. |
| Postmark | A focused transactional-mail workflow where provider-managed templates may be useful | Decide whether a remote template or a repository-owned render is the source of truth; verify the chosen domain and event flow. |
| Resend | A code-first application that wants a compact email integration | Keep receipt semantics outside the transport layer and validate domain plus DKIM behavior against its current docs. |
| SendGrid | A team that may need provider-managed dynamic templates and broader email tooling | Remote editor control can conflict with code review, so establish ownership and rollback rules before launch. |
| Infrai | A small backend that values a self-describing REST surface: discovery exposes schemas and runnable examples in 10 languages. Its single credential also spans 295 routes in 20 modules, which can reduce credential and integration maintenance when email is one small part of the backend. | It is not a fit when email events must arrive through a push webhook, when SMTP relay is required, or when mainland-China email readiness must support a compliance decision. Email feedback is pull-only. |

This is why I would not declare a universal winner. Amazon SES exposes more building blocks than some solo products want to operate. Postmark, Resend, and SendGrid each present different template workflows. A self-describing API reduces integration reading, but polling latency can rule it out for a system whose failover must react immediately; in that case, choose a provider whose documented event mechanism meets the latency requirement. Those are product-shaping limitations, not minor checkbox gaps.

For Infrai specifically, one API key and one bill cover the broader 295-route, 20-module surface. That second advantage matters only if this backend also outsources other undifferentiated capabilities: it means the receipt worker does not add another credential lifecycle or another vendor invoice to reconcile. It is irrelevant for a business that already standardized its mail operations elsewhere. My first instinct is usually to value consolidation; the pull-only event model is the reason I would stop and test the latency requirement before acting on that instinct.

For the gaming receipt in this build, **repository-owned HTML plus a replaceable transport is the default**. I would permit a remotely edited welcome-message body only if the product truly needs non-developer editing, and I would keep authentication-sensitive or purchase-record content under code review. Ship weekly, but make the blast radius small.

## A release gate I can keep running

Before production, verify the custom sending domain, confirm DNS health, send a receipt fixture to controlled mailboxes, and retain the returned message ID with the order and template version. After release, review bounces and complaints at the promised polling interval. Suppress addresses when the selected provider's event semantics require it, and investigate changes rather than repeatedly sending to a known bad destination.

Then repeat the gate after a From-address change, DNS migration, or DKIM rotation. This is mundane work. It is also the work most likely to keep a correct receipt from disappearing between payment settlement and the player's inbox.

## References

- [Amazon SES verified identities](https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html)
- [Amazon SES DKIM authentication](https://docs.aws.amazon.com/ses/latest/dg/send-email-authentication-dkim.html)
- [Postmark domain verification](https://postmarkapp.com/support/article/1090-how-do-i-verify-a-domain)
- [Postmark templates](https://postmarkapp.com/developer/user-guide/templates/templates-overview)
- [Resend domain documentation](https://resend.com/docs/dashboard/domains/introduction)
- [Resend email templates](https://resend.com/docs/dashboard/emails/templates)
- [SendGrid domain authentication](https://www.twilio.com/docs/sendgrid/ui/account-and-settings/how-to-set-up-domain-authentication)
- [SendGrid transactional templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [FTC CAN-SPAM compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
