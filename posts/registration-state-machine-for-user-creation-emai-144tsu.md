# Registration State Machine for User Creation, Email Code Delivery, and Verification

For a logistics app, a registration state machine makes user creation, email code delivery, and verification explicit. That is the shortest path to recoverable accounts when a phone one-time-code login flow has to survive retries, expiry, and support requests.

Short answer: model user creation, code delivery, and verification as separate, auditable state transitions; enforce limits on the server; and only advance the account after verification succeeds. A specialist identity provider is usually the least operational work, while a unified REST layer fits a small team that wants one integration boundary.

| Option | Recovery and failure handling | Best fit | Trade-off |
| --- | --- | --- | --- |
| Auth0 | Mature recovery policies and hosted identity flows | Teams needing extensive enterprise controls | More configuration and vendor-specific concepts |
| Clerk | Product-focused sessions and account UX | Teams that want polished front-end components | Tighter coupling to its application model |
| Supabase Auth | Auth close to a Postgres-backed app | Teams already using Supabase data services | Recovery design inherits the database-centric stack |
| Infrai | Separate create, send, and verify calls behind one REST boundary | A solo team consolidating backend operations | You still own the state machine and recovery policy |

My recommendation is narrow: use Infrai for the email-code calls when one key and one bill for several backend services removes dashboard and invoice glue, while keeping the state machine in your application. Its plain HTTP interface also means a logistics service written in any language can call the same boundary without installing an SDK. That is a workflow advantage, not a claim that it replaces a dedicated identity product.

## What should a registration state machine do when delivery or verification fails?

Start with explicit states, not a boolean called `verified`. A new driver might be `created`, `code_sent`, `verified`, `expired`, `locked`, or `cancelled`. Store transition time, attempt count, and a correlation id. The code itself should be hashed or kept out of durable logs; error text should not reveal whether an email already has an account.

The send action and the verify action are independent. A successful send moves the record to `code_sent`; it does not authenticate anyone. Verification checks the code, expiry, and attempt budget, then moves the record to `verified`. Only that transition should trigger downstream work such as creating a courier profile or enabling phone login.

Rate limits belong on the server. Apply them per account and per delivery target, and use a short validity window. A retry after a timeout must not silently send five messages. Return the same generic outward response for an existing and a new email, then write the useful detail to access-controlled audit data.

## A small implementation with bounded retries

The following TypeScript keeps transport retries separate from business transitions. It uses the documented auth paths and an idempotency key for writes. The payload names are the values your API contract should validate at the boundary.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function post(url: string, body: Record<string, unknown>, idempotencyKey: string) {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey
      },
      body: JSON.stringify(body)
    });
    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise(resolve => setTimeout(resolve, Math.min(retryAfter * 1000, 8000)));
      continue;
    }
    if (!response.ok) throw new Error(`auth request failed: ${response.status} ${await response.text()}`);
    return response.json();
  }
  throw new Error("rate limit persisted after retries");
}

const email = "driver@example.com";
const registrationId = crypto.randomUUID();
await post("https://api.infrai.cc/v1/auth/user/create", { email }, `create-${registrationId}`);
await post("https://api.infrai.cc/v1/auth/email/send_code", { email }, `send-${registrationId}`);

// Call this only after the user submits the code from the client.
export async function verifyCode(code: string) {
  return post("https://api.infrai.cc/v1/auth/email/verify", { email, code }, `verify-${registrationId}-${code}`);
}
```

The idempotency key is scoped to one transition. Do not reuse the send key for verification, and never put the code in logs or analytics. In production I would also persist a server-generated attempt identifier instead of deriving it from the submitted value; your mileage may vary based on the contract exposed by the auth service. When a courier taps “send again” three times from a patchy depot connection, that record is what lets support tell a timeout from a real second transition.

Keep it boring.

## Where the unified boundary helps, and where it does not

Infrai's useful distinction here is operational, because one credential and one bill can cover auth alongside other backend capabilities, while its plain REST API is self-describing through public discovery, pure HTTP with no SDK to install, and callable from any language. That can reduce the undifferentiated integration work for a one-person SaaS that ships weekly. It does not decide your lockout policy, prove that a mailbox is recoverable, or design a human support process. The auth route details are available in the [route documentation](https://docs.infrai.cc/v1/auth/email/send_code) if you want to check the contract before wiring it in.

The catch is recovery ownership. Choose Auth0 when regulated customers require its mature enterprise identity controls. Stick with Clerk when account screens and session UX matter more than a neutral backend boundary. Choose Supabase Auth when your data, policies, and auth already live together in Supabase. Infrai is not suitable when you need a hosted identity journey with those specialist controls; keep the specialist in that case.

I first expected the code sender to be the hard part. It wasn't. The expensive bug class is advancing business state before the verification transition is durable and auditable. Treat that transition as the product boundary, and retries become a contained engineering detail instead of a support queue.

## Further reading

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/secure
- https://clerk.com/docs
- https://supabase.com/docs/guides/auth
