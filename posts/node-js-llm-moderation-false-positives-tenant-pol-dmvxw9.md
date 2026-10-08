# Node.js LLM Moderation False Positives: Tenant Policy Bands that Allow Support Triage

LLM moderation false positives happen when a support-ticket system treats a broad model signal as a final policy decision, so reserve automatic blocking for narrow, high-confidence cases, send ambiguity to review, and record every decision against the tenant that caused the work. The deciding constraint is not the model score by itself. It is the cost of being wrong, multiplied by ticket volume, reviewer time, and the customer relationship hidden behind a short message.

TL;DR: Treat the model output as evidence, not policy. Normalize it into stable categories, apply tenant-aware decision bands, preserve reasons, and meter both inference and review work. False positives happen when context is missing, a broad category is mistaken for a final verdict, or one threshold is applied to tenants with different workflows.

This framing fits a solo SaaS. Ship the small control plane first. Outsource commodity inference behind a generic adapter, but own the policy table, audit record, and cost ledger because those parts encode the business.

## Why do LLM moderation false positives happen under one policy?

Customer-support text is messy by design. A user may quote an abusive message, paste a phishing email for investigation, describe self-harm while asking for account help, or include a security payload that attacked their own site. Keyword overlap is real, yet the author's intent and the operational purpose are different from the quoted material. A classifier that sees only the body can therefore flag the evidence instead of the behavior.

Consider one concrete path. A customer writes a short subject such as "threat received," pastes the threatening paragraph into the body, and asks the support team to identify the sender. The quoted paragraph can dominate the signal even though the ticket is a report, not a threat. Dropping the quote loses the evidence the reviewer needs; treating the entire body as authored speech assigns the evidence to the wrong person. I would preserve the quote as a separate field, assess it, and send the ambiguous relationship between reporter and quoted speaker to review. The trade-off is another structured field and a little more adapter work. It is cheaper than teaching every downstream worker to rediscover who said what.

Context wins.

Context can disappear earlier than expected. Email replies contain signatures and quoted threads. Chat transcripts mix agent and customer turns. Attachments may carry the information that makes a terse sentence harmless. Even normalization can change the input: stripping quotation markers removes a useful signal, while concatenating every historical message can make an old violation look current.

Then policy adds another translation step. A category score answers a narrower question than "what should this application do?" Turning any nonzero signal into a block collapses uncertainty, severity, and operational consequence into one bit. That is where avoidable false positives become customer-facing failures.

Use three decisions, but do not pretend they are equally expensive:

- `allow` continues through ticket triage and still leaves an audit record.
- `review` holds or limits the action while a person sees the relevant context.
- `block` prevents the action and requires a precise, durable reason.

A ticket can also be allowed into the support system while a risky attachment is quarantined. Decisions belong to actions, not vaguely to a whole user. This distinction keeps the policy useful when the same text participates in storage, agent display, automated replies, and outbound email.

## The constraint that changed the build

Per-tenant cost visibility changes the shape of the runtime. A global queue depth cannot tell me which account generated inference calls, which account consumed reviewer minutes, or which policy produced repeated escalations. Without those dimensions, a noisy tenant can tax every other customer and the product owner cannot tell whether automation is buying time or creating a second support job.

The unit of accounting should be a moderation attempt, not merely a ticket. One ticket may be checked on creation, checked again after an edit, and checked before an automated response leaves the system. Give each attempt an immutable ID and attach `tenantId`, `ticketId`, `policyVersion`, model-adapter metadata, final action, and review outcome. Store input hashes rather than copying sensitive text into an analytics table. Keep the protected content in the system that already owns its access controls and retention rules.

This is a revenue-per-hour decision. Detailed metering is worth building because it answers whether a tenant's automation is profitable and where human attention goes. A bespoke classifier is harder to justify early. Weekly shipping favors a replaceable inference boundary plus a small policy layer that can be inspected without retraining anything.

The ledger also exposes a subtle problem: a low inference bill can coexist with an expensive review queue. Track workload in separate columns. Model calls, usage units, queue entries, review duration, reversals, and user appeals describe different costs; blending them into one currency hides the cause.

Keep them separate.

## The smallest working policy loop

Start with a provider-neutral result. The adapter may call a hosted API or a self-hosted service, but the rest of the application should receive the same narrow contract. Categories are application vocabulary. Provider labels get mapped at the edge.

```ts
type RiskCategory = "threat" | "harassment" | "self_harm" | "fraud";
type Action = "allow" | "review" | "block";

interface RiskSignal {
  category: RiskCategory;
  score: number;
  evidence: string[];
}

interface ModerationInput {
  attemptId: string;
  tenantId: string;
  ticketId: string;
  subject: string;
  body: string;
  quotedText?: string;
}

interface PolicyBand {
  reviewAt: number;
  blockAt: number;
}

interface TenantPolicy {
  version: string;
  bands: Record<RiskCategory, PolicyBand>;
}

interface ModerationAdapter {
  assess(input: ModerationInput): Promise<RiskSignal[]>;
}
```

The thresholds are configuration, not universal best practices. Their values must come from labeled examples and the consequence of each action. The invariant is more important: `reviewAt` is lower than `blockAt`, a matched category records its reason, and missing configuration fails into review rather than silently widening an automatic block.

Name an immutable version such as `support-v3` on every result. A bare "current policy" label cannot reproduce yesterday's choice after the bands change.

```ts
interface Decision {
  action: Action;
  reasons: RiskSignal[];
  policyVersion: string;
}

function decide(signals: RiskSignal[], policy: TenantPolicy): Decision {
  let action: Action = "allow";
  const reasons: RiskSignal[] = [];

  for (const signal of signals) {
    const band = policy.bands[signal.category];
    if (!band) {
      return { action: "review", reasons: [signal], policyVersion: policy.version };
    }

    if (signal.score >= band.reviewAt) reasons.push(signal);
    if (signal.score >= band.blockAt) action = "block";
    else if (signal.score >= band.reviewAt && action !== "block") action = "review";
  }

  return { action, reasons, policyVersion: policy.version };
}
```

Short code. Long consequences.

Run the adapter and decision function in a durable job rather than the request that accepts the ticket. The intake path stores the ticket and an outbox event in one transaction. A worker claims that event, performs the assessment with an idempotency key, writes the decision, and advances the ticket. Retries reuse the same attempt ID so a timeout does not become invisible duplicate spend.

There are two exceptions. If the product must prevent content from being displayed before any check, keep the new ticket in a pending state. If the adapter is unavailable, use an explicit policy choice such as pending review; do not turn transport failure into `allow` or pretend it was a content decision.

The audit row should explain enough to replay the choice without depending on mutable configuration. Record the policy version and normalized signals. Record adapter latency and usage units for cost attribution. Do not store a free-form chain of thought. Reviewers need cited text spans, category labels, and the decision rule, while operators need stable fields they can aggregate.

## Operating the queue without letting it own the roadmap

A review queue is a product surface, even if customers never see it. Prioritize by potential harm and waiting time, then group by tenant only where that improves context. Pure first-in, first-out processing can leave a severe item behind a burst of low-risk tickets. Pure risk ordering can starve uncertain but harmless-looking tickets. A bounded aging rule gives old items increasing priority without erasing severity.

Keep the reviewer action small: confirm the category, change the action, choose a reason, and optionally flag a policy gap. Every override becomes labeled feedback. It should not automatically retrain a model, because one hurried click is weak evidence. Accumulate examples, remove duplicates, and review disagreements before changing a band.

For each tenant, watch four ratios over a fixed window: reviews per moderation attempt, blocks per attempt, reviewer reversals per reviewed item, and appeals per communicated restriction. These are operating signals, not a universal scorecard. A rising review ratio can mean the tenant's content changed, a mapping changed, or a threshold moved. The audit dimensions tell you which.

Test policy changes by replaying a versioned, access-controlled set of tickets that includes quoted abuse, threat reports, security samples, short messages, and multilingual content actually supported by the application. Compare the old and proposed actions by category and tenant. Release the policy independently from application code, but require validation that every category has ordered bands and that a version cannot be edited after use.

Privacy and regional obligations need legal review tied to the real product, users, and data flows. The engineering boundary can still be clear: minimize copied content, set retention by data class, restrict reviewer access, log access, and make deletion propagate to derived stores where required. Do not encode "US" or "EU" as magic threshold presets. Geography alone does not describe a tenant's policy, user age, contractual promises, or the action being moderated.

## What I would change at scale

The first upgrade would be workload isolation. Put tenant-aware quotas on moderation attempts and review submissions, then use fair queue scheduling so one import cannot consume every worker. This protects latency and makes capacity planning legible. It also turns abuse controls into explicit policy instead of emergency queue surgery.

Queues aren't free.

Next, separate initial screening from review prioritization. A ranking component can order already-reviewable items using the ticket, policy reason, severity, and age. It must not silently convert a review into a block. Ranking is a scheduling aid; the policy engine remains the authority.

At higher volume, shadow a proposed adapter or policy version on sampled traffic. Write its output beside the active result without changing the user-visible action. Compare category mappings, disagreement rates, queue load, and estimated reviewer demand per tenant before promotion. Sampling must be recorded so the comparison denominator is honest.

I would keep the generic boundary even after adding more capacity. Gateway projects can centralize access to several model backends, while reranking systems can score the relevance of candidate records to a query. Those are useful component patterns, but neither supplies the application's moderation policy. The durable asset is the decision history connected to tenant economics.

The trade-off is extra state. Versioned rules, idempotent attempts, and immutable audit rows take more work than `if (score > threshold) block()`. They pay for themselves when a customer asks why a ticket disappeared, when a reviewer queue spikes, or when an adapter changes. For a one-person SaaS, that answerability protects shipping time.

## Decision rule

Use automatic blocking only when the category is specific, the evidence is present, the action's harm is understood, and replayed examples support the chosen band. Route uncertainty to a capacity-managed review queue. Allow the rest, while retaining enough structured evidence to detect drift.

Keep costs attributable to the tenant and the policy version. That makes threshold changes business decisions rather than aesthetic arguments about scores. It also keeps the model in its proper role: a replaceable source of signals inside a system that owns its actions.

## References

- https://docs.cohere.com/docs/rerank-overview
- https://github.com/BerriAI/litellm
