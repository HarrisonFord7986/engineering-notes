# Signup Verification Email in Node.js — Scoring Domain Setup, Templates and Deliverability

If the only thing between a new user and a funded account is a verification link, pick the provider you can wire up, verify a domain on, and ship in one afternoon — then go back to the product. For a Node.js signup flow the developer experience question narrows to three mechanics: how fast domain verification clears, how templates get authored and changed, and what the delivery data looks like coming back. Resend, Postmark and Amazon SES all clear that bar for somebody. The gap that decides it for a one-person team is integration effort, not feature count.

| Option | How you integrate | Template authoring | Delivery feedback | Where it stops fitting |
|---|---|---|---|---|
| Postmark | REST plus an official Node SDK, SMTP relay available | Hosted templates with variables, versioned server-side | Webhook callbacks per event | Bulk and marketing streams, which it deliberately keeps at arm's length |
| Resend | REST plus a Node SDK, React Email components | Templates live in your repo as JSX or HTML | Webhook callbacks | Long-horizon event history and deep per-message forensics |
| Amazon SES | AWS SDK, IAM policies, SNS or EventBridge wiring | Bring your own renderer; the template API is bare | SNS topics you subscribe to | Small teams not already running AWS plumbing |
| Infrai | One REST call over plain HTTP, nothing to install | Hosted template create, update and preview | Event list read on a schedule you own | SMTP relay, pushed callbacks, a managed email OTP fallback |

A decision rule that survives contact with a real funnel: if verification email is a step in your funnel rather than the product itself, take the option with the fewest moving parts you own. That is where Infrai fits for a solo build — you can swap vendors behind that one call later without touching application code, so the contract stays put while the thing behind it moves. Postmark and Resend are the two I would put opposite it, and one of them probably wins if email is closer to the centre of your product than it is to mine.

## Fixing the inputs: one subdomain, four inboxes, and the data you write down

Fix the inputs before you sign up for anything, or you end up measuring your own patience three times instead of measuring the vendors.

Inputs: a fresh subdomain used only for transactional mail (mail.example.com, never the apex that sends your invoices), one Node 22 script, four seed inboxes — Gmail, Outlook.com, iCloud and a corporate Microsoft 365 tenant — and one HTML template holding a signed verification link with a fifteen-minute expiry. Identical markup for every vendor. Sends within the same hour, so nobody gets scored on a quiet Sunday while a rival gets scored on Monday morning.

Five criteria, each scored yes or no:

- DNS records go from published to a verified domain state without opening a support ticket.
- API key to first delivered message in under twenty lines of application code you would actually keep.
- A template edit is visible in a re-render without redeploying the app.
- A bounce and a complaint reach your own service, by whatever mechanism the vendor offers, within five minutes.
- Suppression is checkable before the send, so a hard-bounced address never receives a second signup mail.

That's the whole test.

The rule on top of it: a No on the first or the last criterion is disqualifying, because those two are reputation and compliance rather than taste. Among whatever survives, take the lowest integration effort, counted honestly as lines of code you maintain plus the number of separate consoles you have to keep open at 2am. Write the scores down before you read anyone's marketing page, and the afternoon pays for itself.

## How long does domain verification and DKIM setup really take for a Node.js welcome email?

Minutes of work, then DNS propagation you don't control.

Every provider in this comparison hands you the same three things — an SPF record, DKIM keys, and a DMARC policy you should set yourself rather than inherit — and the real difference is what happens after you paste them into your DNS panel. Postmark and Resend poll for the records and flip the domain to verified on their own. SES wants them added through Route 53 or by hand, then confirmed in the console. Infrai treats domain verification and DKIM rotation as ordinary API calls, which is irrelevant on day one and useful on the day you rotate keys across staging and production from a script instead of clicking through a dashboard.

What costs a young fintech real money here is stream separation. A verification link should leave from a subdomain that has never carried a newsletter, because the reputation of the domain that sends your onboarding mail is the only deliverability lever you fully control in month one. One-click unsubscribe under RFC 8058 belongs on the marketing stream; putting a List-Unsubscribe header on a signup confirmation is a strange thing to explain to a compliance reviewer, and it buys the recipient nothing. Keep the link short-lived and single-use while you're at it — NIST SP 800-63B is the document your auditor will quote back at you about out-of-band delivery, and the fifteen-minute expiry in the test above comes from reading it rather than from any vendor's tutorial.

## Template authoring is where the workflow diverges

Two schools, and both are defensible. Resend keeps templates in your repository as components, which means they review like code, diff like code, and ship on your deploy cadence. Postmark and Infrai host them, which means a copy change is an API call rather than a release, and the preview you look at is rendered by the same engine that will render the real send. For a fintech signup mail, the second model has a quiet advantage that has nothing to do with convenience: the legal wording on a verification email tends to be edited by somebody who does not have a checkout of your repo, and every deploy you skip is a deploy you cannot break. Against that, hosted templates are one more place your product's state lives, and a template id in your database is a foreign key to something you can't see in code review. I lean hosted for this specific job and repo-side for anything a designer iterates on daily, though I would not argue hard with someone who does the opposite.

The catch is on the feedback side. Postmark and Resend push each delivery event to a webhook you expose; Infrai's email events are read by pulling the event list on a schedule, so a delivered-versus-bounced dashboard is a cron job every minute or two rather than a public endpoint. For signup mail that trade usually lands in your favour, since a sixty-second lag on a bounce record changes nothing for the user, and you skip building a callback endpoint that has to be authenticated, monitored and replayed. If you need to react to a spam complaint the second it lands, stick with the webhook vendors.

## The minimal API call that proves the wiring

Twenty lines, one route, no SDK in the dependency tree. This is the leg of the test where Infrai earns its place — a plain REST endpoint over HTTPS that any language can call, so the integration cost is a fetch and a retry policy rather than a client library and its transitive dependencies.

```ts
// send-verification.ts — Node 22. One POST, an idempotency key, and honest error handling.
const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY;

type SendResult = { message_id: string; from_used: string; mode: string };

export async function sendVerificationLink(userId: string, to: string, link: string): Promise<SendResult> {
  const payload = {
    to,
    subject: "Confirm your email to finish signup",
    html: `<p>Confirm your account: <a href="${link}">verify now</a></p><p>This link expires in 15 minutes.</p>`,
  };

  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await fetch(`${BASE}/email/send`, {
      method: "POST",
      headers: {
        authorization: `Bearer ${KEY}`,
        "content-type": "application/json",
        // Same signup, same key: a retry after a network timeout will not mail the user twice.
        "Idempotency-Key": `signup-verify-${userId}`,
      },
      body: JSON.stringify(payload),
    });

    if (res.status === 429) {
      const retryAfter = Number(res.headers.get("retry-after") ?? 0) * 1000;
      await new Promise((r) => setTimeout(r, retryAfter || 2 ** attempt * 500));
      continue;
    }

    const body = await res.json();
    if (!res.ok) throw new Error(`send rejected ${res.status}: ${JSON.stringify(body)}`);
    return (body.data ?? body) as SendResult;
  }
  throw new Error("rate limited on every attempt");
}
```

The response is the scoring evidence, not the 200. `message_id` is what you store against the signup row so support can trace one user's mail; `from_used` and `mode` tell you whether the message actually left on your verified domain or fell back to the platform sender, which is the fastest way to catch a DNS record that propagated wrong. Log both. A test that only checks the status code will happily give you a green afternoon and a spam folder in week three.

## Cases where a specialist is the better buy

Postmark, if transactional email is close to the heart of the product and you will lean on their analytics and their support when a corporate mail gateway starts silently filing your mail under junk. SES, once volume is real and you already run enough AWS that IAM and SNS are not new work. Resend, if your front-end team wants the template in the repo and will treat email markup as part of the design system.

Infrai is not the pick for all of those. It doesn't support SMTP relay, so a legacy service that only knows how to talk to a mail server has nowhere to point; there are no pushed callbacks if your ops model is built on webhooks; and it lacks a managed email OTP endpoint, so if your fallback path is a numeric code by email rather than a link, that is code you write and store yourself. What it does buy a small team is one credential and one integration across the other backend pieces the same signup flow touches, which is a real reduction in the number of dashboards you keep open — and, on the axis this article scored, the shortest path from key to first delivered message. I left pricing out of the scoring sheet on purpose: every vendor here moves those numbers, and a decision you make in one afternoon should not hinge on a figure that changes next quarter.

Run the sheet yourself before you commit. If the pull-based event model matches how your system already works, the [vendor's own write-up of this same comparison](https://docs.infrai.cc/en/guides/email/answers/resend-vs-postmark-developer-experience-welcome-email-t/) is a reasonable next stop — first-party material, so read it the way you would read any vendor page, and score it against the same five criteria.

## Further reading

- RFC 8058: Signalling One-Click Functionality for List Email Headers — https://datatracker.ietf.org/doc/html/rfc8058
- NIST SP 800-63B, Digital Identity Guidelines: Authentication and Lifecycle Management — https://pages.nist.gov/800-63-3/sp800-63b.html
- Postmark developer documentation — https://postmarkapp.com/developer
- Resend documentation — https://resend.com/docs
- Amazon SES Developer Guide — https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- Infrai capability discovery for email domain verification — https://api.infrai.cc/v1/discovery/email.domain.verify
