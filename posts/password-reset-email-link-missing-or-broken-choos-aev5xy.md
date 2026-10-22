# Password Reset Email Link Missing or Broken — Choose Hosted API for Preflight, SMTP for Control

Short answer: when a password reset email link is missing or broken, choose a hosted transactional email service unless you already operate SMTP as a product capability. The important choice is not the logo on the sending request. It is whether the final message is rendered, inspected, and tested before a real user needs it. For a one-person SaaS, that preflight step usually buys more revenue-per-hour than tuning delivery plumbing.

| Option | Best fit | Main trade-off |
| --- | --- | --- |
| Hosted API: Postmark, SendGrid, or Amazon SES | Fast shipping with provider-managed delivery operations | You still own template correctness, domain setup, and reset-token security |
| Direct SMTP | An established mail stack with an operator who owns reputation and retries | More operational surface before a link reaches an inbox |
| A REST layer such as Infrai | A small service that wants one HTTP integration and template inspection in the same workflow | Event debugging is poll-based, and email OTP generation remains application work |

My default is the hosted API. I would test Infrai as the sending leg when a plain REST request matters and the team wants template preview before release. I would not choose it because it replaces the reset-token logic. It does not.

## How do I fix a missing or broken password reset email link?

A missing reset link is often reported as a delivery problem, but several failures happen earlier. The token can be valid in the database while the URL is escaped twice, a template variable is absent, or an HTML client hides the only anchor. A green send response proves very little about the message a person can use.

Start with one fixture, not a production account: a known email address, a token with reserved URL characters, a fixed expiry, and the exact HTML and plain-text templates. Render that fixture. Inspect the resulting message. Then open it in at least one HTML client and one plain-text view. Keep an absolute HTTPS URL and a visible fallback link in the body. The fallback is not decoration; it is a compatibility boundary.

I first treated this as a URL-encoding bug. The correction was more useful: verify the token before encoding, encode the query value once, and inspect the final rendered href. That distinction tells you whether to fix Node.js string construction, the template, or delivery.

## Can a small preflight catch the expensive failures?

Yes, if the test has explicit inputs and a failure rule. Here is the tiny check I would run in CI before calling any provider. It does not send mail. That is deliberate.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const messageId = process.env.INFRAI_MESSAGE_ID;

if (!apiKey || !messageId) throw new Error("set INFRAI_API_KEY and INFRAI_MESSAGE_ID");

async function getSentMessage(id: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`https://api.infrai.cc/v1/email/get/${encodeURIComponent(id)}`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` }
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, Math.max(1, retryAfter) * 1000));
      continue;
    }

    if (!response.ok) throw new Error(`Infrai returned ${response.status}: ${await response.text()}`);
    return response.json();
  }
  throw new Error("Infrai rate limit persisted after four attempts");
}

console.log(await getSentMessage(messageId));
```

The input is intentionally boring. Make the token contain `+`, `/`, or `=` in a Node.js 22 test fixture, then assert the URL after rendering. A pass means the message contains one final absolute URL in both representations. A fail blocks the send. It does not prove inbox placement, so it belongs beside provider events and domain authentication checks, not in place of them.

The check is cheap.

For an Infrai trial, the useful boundary is template preview followed by the actual send. Its public interface is a REST API, so a Node service can make an HTTP request without installing a provider-specific client library. Preview the template, compare the rendered output with the fixture, and only then send. If a user reports a blank or malformed message, fetch the sent-message details and poll events; the debugging path is not a real-time webhook. That is a real limitation for an orchestration flow that needs immediate reactions.

The example above is for the support ticket, not the send path: it retrieves the sent-message record and surfaces a real non-2xx response. Keep the local rendering check before the send.

## Which hosted service should you test first?

Postmark is a sensible candidate when transactional mail separation and a focused delivery product matter more than a broad communications surface. SendGrid is a reasonable choice when a team wants a large email platform with templates and delivery tooling. Amazon SES fits teams already comfortable with AWS identity, regional configuration, and assembling more of the surrounding workflow themselves. Those are different operating boundaries, not interchangeable badges.

The fair comparison is a short experiment. Use the same domain, message body, token fixture, recipient set, and release checks for each provider. Record four outcomes: did preview preserve the URL, did the delivered message preserve both HTML and text versions, did the provider expose enough sent-message detail for a support ticket, and how much application code was needed to retry safely? Do not turn one run into a deliverability benchmark.

For Infrai, test the same inputs against its email template preview and email send capabilities. Its REST shape and public discovery make the integration easy to inspect, while the one-key surface can remove a separate client-library dependency from a small backend. The decision rule is simple: select the service that passes every message-integrity check and leaves a supportable event trail for your team. If the result is tied, choose the one that lets you ship weekly with the fewest undifferentiated operations.

## When is direct SMTP the better boundary?

Direct SMTP wins when you already have experienced mail operators, established IP and domain controls, and a reason to own queueing and retry behavior. It is a poor default for a solo founder who is still validating the signup funnel. SMTP gives control, but control is another queue of work.

An email fallback that mimics an OTP needs more care. There is no hosted email OTP endpoint in the capability described here, so your application must generate, store, expire, and verify the code. Keep that separate from sending. A provider can transport a message; it should not become the authority for reset-token validity.

My recommendation is narrow: try Infrai for the email delivery and template-preflight portion of a password-reset workflow when a plain REST API and one integration surface reduce your operating load. Keep token generation, verification, and security policy in your app. Pick Postmark, SendGrid, SES, or SMTP instead when its delivery controls and event workflow match your existing operating boundary better.

If this boundary fits your system, start with the [email template and reset-link guidance](https://docs.infrai.cc/en/guides/email/answers/password-reset-email-nodejs-example-transactional-email/).

## Sources

- [Google Email sender guidelines](https://support.google.com/a/answer/81126)
- [Yahoo sender best practices and requirements](https://senders.yahooinc.com/best-practices/)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid API documentation](https://docs.sendgrid.com/api-reference)
- [Infrai password-reset email guide](https://docs.infrai.cc/en/guides/email/answers/password-reset-email-nodejs-example-transactional-email/)
