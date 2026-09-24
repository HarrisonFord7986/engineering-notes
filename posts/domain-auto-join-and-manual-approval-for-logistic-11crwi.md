# Domain Auto-Join and Manual Approval for Logistics Partner Workspaces

Use domain auto-join only after an organization administrator has verified control of the domain, and only for a low-privilege member role. Keep manual approval for public email domains, shared partner domains, privileged roles, and ambiguous ownership. That rule gives a logistics SaaS fast onboarding without treating an email suffix as authorization.

| Enrollment path | Good default when | Main control | Operational cost |
| --- | --- | --- | --- |
| Verified-domain auto-join | One company controls the domain and new members start with narrow access | Domain proof, exact match, safe default role | Exceptions and domain lifecycle |
| Manual invite approval | Identity or tenant membership needs human context | Named approver and expiring invite | Queue time and support work |
| Hybrid | Partner networks mix clear and ambiguous domains | Policy chooses a path per request | More states to test |

**Short answer:** choose the hybrid. Auto-join verified corporate domains into a restricted workspace role; route every exception to an auditable approval queue. The extra branch is cheaper than either reviewing every warehouse user or repairing a cross-tenant enrollment mistake. For a one-person SaaS that ships weekly, the winning control is the one that removes routine tickets while keeping irreversible authority out of the fast path.

## Should domain verification trigger auto-join or manual invite approval?

A DNS challenge can establish that someone controls the relevant DNS namespace. It does not prove that every mailbox at that domain belongs in one tenant, that the person still works for the company, or that the person should see shipment manifests. Those are separate claims.

This distinction gets sharp in logistics. A carrier may use one email domain across regional subsidiaries, contractors, and customer-facing teams. Two workspaces may both present plausible claims to it. An acquired company can also keep its old mail domain while its data access moves elsewhere. Automatic enrollment must stop on conflict rather than picking the oldest or newest claimant.

Count the claims: `0` means review, `1` may qualify, and `2+` means review.

Treat verification as a time-bound organization configuration, not a permanent fact. Store the normalized domain, verification method, verifier, timestamp, current status, and tenant identifier. Re-check ownership after an administrative transfer or a deliberate policy interval. Never auto-join on a subdomain because its parent was verified unless that inheritance rule was explicitly approved.

Email lookup needs equally dull rules: trim surrounding whitespace, use the domain portion after the final `@`, normalize the domain with a standards-aware library, and compare exact canonical values. Suffix checks are a trap. `driver@notexample.com` must never match `example.com`.

The authentication boundary matters here. OWASP recommends generic authentication error responses so an outsider cannot use response differences to enumerate accounts. Apply the same idea to enrollment: the public response can say that the request is pending while the internal state records whether the domain was unknown, contested, or awaiting an approver.

## Two criteria carry most of the decision

The first is the **blast radius of a false join**. A new member who can only complete a profile and request depot access creates a recoverable review task. A new member who can export delivery addresses, manage billing, invite peers, or administer single sign-on creates a security incident. Auto-join should therefore assign one fixed, minimal role. It should never copy privileges from the inviter, infer access from a job title, or satisfy a privileged-role request. This is an explicit trade-off: a legitimate dispatcher may wait for approval, because accelerating that edge case isn't worth granting dispatch authority from a domain match.

The second is **the quality of the ownership signal**. A unique corporate domain with one active, uncontested tenant claim is a useful routing signal. A public mailbox provider, shared industry domain, unverified domain, or domain claimed by multiple tenants is not. Manual approval earns its cost in those cases because a human can check a contract, depot roster, or partner record that the authentication system cannot see.

Price is a weak deciding factor. Support minutes do matter to a solo operator, but the revenue-per-hour calculation has to include incident response and deletion work. Automate the high-volume, reversible case. Outsource undifferentiated email delivery and DNS resolution behind narrow interfaces, while retaining the enrollment policy and audit trail in application code.

The hybrid model has a real limitation. It needs a review queue, escalation ownership, domain-claim maintenance, and tests for both paths; a tiny service with five known partner administrators may spend more time maintaining that machinery than approving invitations. In that case, manual approval is the better system until request volume makes the queue a shipping constraint. At the opposite extreme, a workforce directory with authoritative organization membership may support more automation, but the application still has to set its own role boundary. There isn't one path that wins at every scale.

## Make enrollment a state transition, not a side effect

The implementation should decide before it mutates membership. The example below keeps the policy small enough to test and makes every non-routine outcome explicit.

```ts
type Role = "restricted_member" | "dispatcher" | "workspace_admin";

type DomainClaim = {
  tenantId: string;
  domain: string;
  status: "verified" | "pending" | "revoked";
};

type JoinRequest = {
  requestId: string;
  emailDomain: string;
  requestedRole: Role;
};

type Decision =
  | { kind: "auto_join"; tenantId: string; role: "restricted_member" }
  | { kind: "manual_review"; reason: string };

export function decideEnrollment(
  request: JoinRequest,
  claims: readonly DomainClaim[],
  blockedDomains: ReadonlySet<string>,
): Decision {
  const domain = request.emailDomain.toLowerCase();
  const matches = claims.filter(
    (claim) => claim.domain === domain && claim.status === "verified",
  );

  if (blockedDomains.has(domain)) {
    return { kind: "manual_review", reason: "non-corporate-domain" };
  }

  if (matches.length !== 1) {
    return { kind: "manual_review", reason: "domain-claim-not-unique" };
  }

  if (request.requestedRole !== "restricted_member") {
    return { kind: "manual_review", reason: "privileged-role-request" };
  }

  return {
    kind: "auto_join",
    tenantId: matches[0].tenantId,
    role: "restricted_member",
  };
}
```

Keep the database write in a transaction with a uniqueness constraint on tenant membership. Record the request ID, policy version, matched claim, decision, and resulting membership ID in an append-only audit event. A retry can then return the original result instead of creating a second membership or sending duplicate notices.

Test the ugly rows, not just the happy one: two verified claims for the same domain, a revoked claim between decision and commit, mixed-case input, a privileged role request, and two simultaneous submissions. Also test that the external response does not reveal whether a workspace exists. These cases are small. Missing one is expensive.

Deployment deserves a brake. Start in shadow mode, where the policy records the decision but humans still approve. Review disagreements, then enable automatic membership for one restricted role. Track counts of auto-joins, manual-review reasons, claim conflicts, approval latency, reversals, and session revocations. Raw email addresses do not belong in metric labels.

## When is manual approval the better default?

Manual approval wins when a workspace represents a legal entity but its domain represents a broader group; when temporary drivers use agency addresses; when access includes personal delivery data; or when an administrator is being added. It also wins during a migration off a managed identity provider until tenant mappings, session ownership, and deletion semantics have been reconciled.

Migration is where hidden coupling surfaces. Exported users may identify an organization differently from the application database. Existing sessions may live in more than one store. Before enabling auto-join, map every legacy tenant identifier to exactly one workspace and reject ambiguous rows. Dual-running enrollment writers creates split authority, so designate one system as the membership writer while the other remains read-only during the cutover.

Fast onboarding can wait for this.

One writer. Two enrollment paths. Every extra state has to earn its upkeep.

## Deletion must close the same lifecycle

GDPR erasure is not just a row deletion, and Article 17 includes conditions and exceptions that require a documented retention decision. Separate authentication identity, workspace membership, operational records, and legally retained records. The deletion workflow can then revoke access immediately, delete or anonymize eligible data, retain only what has a defined basis, and emit an audit result without pretending every record has identical treatment.

For an account deletion request, mark the identity as deletion-pending first and block new logins. Revoke all server-side sessions, refresh credentials, API tokens, recovery links, and outstanding invitations. Expire browser cookies as defense in depth; RFC 6265 explains cookie expiry, but clearing a browser cookie alone cannot invalidate a server-side session. Then remove memberships and process downstream data through idempotent jobs keyed by one deletion request ID.

This closes a subtle onboarding gap: an email address from a deleted account must not inherit its former sessions or privileges if it later signs up again. Re-enrollment is a fresh request evaluated against the current domain policy. No shortcuts.

The durable decision is therefore narrow: verified-domain auto-join for reversible, low-privilege membership; human approval everywhere context or authority matters. It ships quickly, remains explainable during an audit, and survives provider migration because the policy belongs to the application rather than an authentication vendor.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- https://www.rfc-editor.org/rfc/rfc6265
- https://eur-lex.europa.eu/eli/reg/2016/679/art_17/oj
