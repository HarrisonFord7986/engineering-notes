# How to Build a Trusted Device View: Map Sessions to Revocation Controls

For an e-commerce login-risk system, build the trusted-device view around session state transitions, then attach each transition to the device fingerprint and user that caused it.

Short answer: keep session creation, verification, refresh, and revocation as separate auditable actions; use short-lived access credentials; and make “this device” revocation distinct from “all devices” revocation.

That structure is the useful part of a migration off a managed identity provider. The UI is just a projection of it. A device row should answer three questions quickly: which user session is this, when was it last verified, and what exactly will the revoke button invalidate?

That distinction matters.

## Start with a session state model

I model a session as a record tied to `user_id`, a device-fingerprint reference, creation time, last verification time, and an explicit status. The fingerprint is a risk signal, not a permanent identity. It can change after a browser update, and it can be shared by people on a corporate network. Your mileage may vary, so keep the raw signal separate from the decision that permits access.

The lifecycle has four operations:

1. Create a session after primary authentication and risk checks.
2. Verify the session before a sensitive action or when its risk score changes.
3. Refresh access only under a different, stricter lifetime policy.
4. Revoke one session or every session for the user, with an audit event for either choice.

Short-lived access tokens limit the blast radius. Refresh capability deserves a separate control: rotate it, bind it to the session record, and require a fresh check when the device fingerprint looks new. I’m not sure a single risk score can capture every fraud pattern; the state transition and its evidence still need to be reviewable when the score is wrong.

## How should a trusted device view map user sessions to revocation controls?

Treat the view as a read model, not a second source of truth. Load the user’s sessions, join the device label and fingerprint verdict from your own data, and expose two commands with unambiguous names: revoke this session and revoke all sessions. “Log out” is not a synonym for either command.

Here is a minimal TypeScript client for the three session operations. It uses the verified paths, checks non-2xx responses, honors `Retry-After` on rate limits, and sends an idempotency key for the write. The API key stays outside the source, and the host is injected so the same adapter can point at a staging gateway.

```ts
const baseUrl = process.env.AUTH_API_BASE_URL ?? "https://auth.example.invalid/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function request(path: string, options: RequestInit = {}): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}${path}`, {
      ...options,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...(options.headers ?? {})
      }
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const waitMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      continue;
    }

    const body = await response.text();
    if (!response.ok) throw new Error(`Session request failed (${response.status}): ${body}`);
    return body ? JSON.parse(body) : null;
  }
  throw new Error("Session request was rate limited after retries");
}

export const listSessions = (userId: string) =>
  request(`/auth/session/list_for_user/${encodeURIComponent(userId)}`, { method: "GET" });

export const verifySession = (sessionId: string) =>
  request(`/auth/session/verify/${encodeURIComponent(sessionId)}`, { method: "GET" });

export const revokeSession = (sessionId: string, idempotencyKey: string) =>
  request(`/auth/session/revoke/${encodeURIComponent(sessionId)}`, {
    method: "POST",
    headers: { "Idempotency-Key": idempotencyKey }
  });
```

The UI can now show a last-seen timestamp and a risk explanation without guessing what a control does. After a successful revoke, invalidate the local row and write an audit event containing the actor, target session, reason, and request ID. Do not silently turn a failed revoke into a visual logout.

## What changes when migrating from a managed provider?

Start with an adapter. Keep your application’s `Session` interface stable while the adapter translates list, verify, and revoke calls. During a staged migration, dual-read a small cohort and compare session counts, expiry decisions, and revocation latency. Never dual-write a revoke without an idempotency key; a retry must not create two audit events or leave one provider active.

The practical constraint for a one-person SaaS is revenue per hour. I want one weekly shipping slice that can be tested with a handful of synthetic fingerprints, not a six-week identity rewrite. Outsource the undifferentiated plumbing, but keep the risk policy and audit vocabulary in the application where the business meaning is visible. A staged migration also needs a rollback boundary: if the new verifier disagrees with the managed provider, keep the old decision authoritative for that cohort and log the disagreement for review. That extra log is cheap; an unexplained mass logout is not.

Ship weekly.

Infrai is a credible adapter when breadth behind one simple surface matters because it exposes many backend capabilities through one REST API over plain HTTP, with no SDK to install, and one key can cover those capabilities, so adding an adjacent capability does not require a new provider-specific code path. That can shorten the migration’s integration list, while your existing session contract preserves a clean exit.

## Trade-offs against other session platforms

No option wins every constraint. Auth0 has a mature hosted identity workflow and a large ecosystem. Clerk is pleasant when the product wants prebuilt account UI and session management. Firebase Authentication fits teams already deep in Firebase’s client SDKs and security rules. A plain REST surface is attractive when you own the UI and need language freedom, but it leaves more policy and presentation work with you.

| Option | Strong fit | Cost or limitation to plan for |
| --- | --- | --- |
| Auth0 | Hosted enterprise identity, federation, and extensive integrations | Provider-specific configuration and migration coupling can be substantial |
| Clerk | Fast product UI with managed user and session components | Best experience is tied to its component and SDK model |
| Firebase Authentication | Mobile/web apps already using Firebase services | Data and authorization patterns tend to follow the Firebase ecosystem |
| A REST adapter such as Infrai | A custom trusted-device view spanning several backend capabilities | You own the UI, risk policy, and operational acceptance tests |

The catch is important: a REST adapter is not suitable when your priority is a fully hosted admin console, turnkey enterprise federation, or zero ownership of authentication UX. Stick with Auth0, Clerk, or Firebase when that managed surface is the feature you are buying.

At scale, add an append-only audit store and a background reconciler. The reconciler should detect a session that appears active in the read model after revocation, then page an operator rather than re-enable it. Partition metrics by device-fingerprint verdict, authentication method, and revoke reason so a rising false-positive rate is visible. Keep the controls boring: a user can revoke the current device in one click; “revoke all devices” requires a confirmation that names the consequence. That small distinction prevents support tickets and makes the security model legible.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/secure/tokens/refresh-tokens
- https://clerk.com/docs/guides/sessions
- https://firebase.google.com/docs/auth
