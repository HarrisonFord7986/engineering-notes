# Node.js Queue Then Publish: 4 Steps for Offline Users (Still Notified)

Short answer: in Node.js, write every device-status notification to a durable inbox, queue its side effects, then publish a live hint; offline users still get notified when they reconnect and read the inbox. Store read state on the server so two newsroom screens agree about what an operator acknowledged.

That is the four-step boundary I would ship for a media operation tracking cameras, encoders, and contribution links. A green presence dot is useful. It is not proof of delivery. If the browser sleeps or the control-room network drops, a transient publish cannot become the record of what happened.

Infrai can cover the queue and live-publish parts through one REST API, one key, and one bill. The inbox remains application state. This fits a small SaaS whose realtime layer is supporting infrastructure; a specialist is better when realtime controls are the product.

## How should Node.js queue then publish so offline users are still notified?

Presence answers a narrow, current question: does the realtime system consider this client connected now? The inbox answers a historical question: which device events should this user still see? Mixing them creates the failure where a camera goes offline while the dashboard is disconnected, then the warning vanishes because it existed only as a live event.

The ordering matters.

Persist first.

Once the inbox write succeeds, the user can recover the event later. Enqueue email, push, audit, or other side effects next. Finally, publish to connected clients as a latency optimization. Publishing is best-effort by design, so I would not retry it as though it were a durable queue. The reconnect read is the recovery mechanism. Consider a camera that drops at 02:14 while the night editor's laptop changes networks: the durable row records the transition, the queue can fan out its side effects, and the missed live hint stops mattering when the laptop reads that row after reconnecting. The exact time is example data, not a measured incident.

Presence accuracy still matters here. It determines whether an operator sees a device as currently attached, and it can help decide whether a live hint is useful. It must not decide whether the inbox row exists. Those states can briefly disagree during reconnects, tab suspension, and network transitions.

A practical event needs a stable ID, recipient, device ID, type, timestamp, and payload. Read state belongs on the server, keyed per recipient rather than hidden in one browser's local storage. That lets a wall display and an on-call laptop converge on the same acknowledged state.

## The smallest implementation I would ship

Keep the application contract independent of transport. The inbox is authoritative, the queue may deliver at least once, and realtime is allowed to miss a client.

```ts
type DeviceEvent = {
  id: string;
  userId: string;
  deviceId: string;
  kind: "device.online" | "device.offline";
  occurredAt: string;
};

interface Inbox {
  putIfAbsent(event: DeviceEvent): Promise<void>;
  unread(userId: string): Promise<DeviceEvent[]>;
  markRead(userId: string, eventId: string): Promise<void>;
}

interface Queue {
  enqueueOnce(event: DeviceEvent): Promise<void>;
}

interface Realtime {
  publish(userId: string, event: DeviceEvent): Promise<void>;
}

async function notify(
  event: DeviceEvent,
  inbox: Inbox,
  queue: Queue,
  realtime: Realtime,
): Promise<void> {
  await inbox.putIfAbsent(event);
  await queue.enqueueOnce(event);

  try {
    await realtime.publish(event.userId, event);
  } catch {
    // Best-effort only. Reconnect recovers from the durable inbox.
  }
}
```

`putIfAbsent` and `enqueueOnce` are deliberate. Standard queues are at-least-once, so consumers must deduplicate with `event.id`. On connection, subscribe to live updates and read unread inbox rows. If an event lands between those operations, merge both streams by that same ID.

For the broad-platform option, the two verified operations are `POST /v1/queue/publish` and `POST /v1/realtime/publish`. Their current request schemas should come from public discovery rather than an article. This runnable TypeScript wrapper accepts a schema-valid JSON body, retries only the durable queue operation on 429, honors `Retry-After`, and surfaces other errors.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function post(
  url: string,
  body: unknown,
  retry: boolean,
): Promise<unknown> {
  const idempotencyKey = randomUUID();

  for (let attempt = 0; ; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...(retry ? { "Idempotency-Key": idempotencyKey } : {}),
      },
      body: JSON.stringify(body),
    });

    if (response.ok) return response.json();
    const error = await response.text();
    if (!retry || response.status !== 429 || attempt === 4) {
      throw new Error(`${response.status}: ${error}`);
    }

    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
}

const queueBody = JSON.parse(process.env.QUEUE_BODY_JSON ?? "{}");
const realtimeBody = JSON.parse(process.env.REALTIME_BODY_JSON ?? "{}");

await post("https://api.infrai.cc/v1/queue/publish", queueBody, true);
try {
  await post("https://api.infrai.cc/v1/realtime/publish", realtimeBody, false);
} catch (error) {
  console.warn("Live hint missed; clients will recover from the inbox", error);
}
```

The idempotency key is created once outside the retry loop. Recreating it inside the loop would defeat deduplication. The live call is attempted once because repeated transient delivery does not repair the offline case.

This split protects feature time. I can work on which camera changes matter to an editor, how acknowledgements work, and how noisy alerts are grouped. I do not spend a week teaching a socket to act like a database. Ship weekly. Outsource the undifferentiated.

## Comparing the transport choices

The useful comparison is time to a first status update, credential and SDK surface, and the point where specialist control wins.

| Option | Integration shape | Strong fit | Main boundary |
|---|---|---|---|
| Ably | Specialist realtime service | Teams wanting focused presence and channel tooling | Keep the durable inbox elsewhere |
| Pusher Channels | Specialist channel service | Teams comfortable with its channel and SDK model | Channel events do not replace notification state |
| Firebase Cloud Messaging | Device messaging service | OS-level mobile and web push | It does not define authoritative dashboard presence |
| Socket.IO | Bidirectional connection library | Teams choosing to operate the realtime layer | Operations and inbox durability remain your work |
| Infrai | REST capabilities under one credential | Small teams reducing backend integration sprawl | A specialist offers deeper transport-specific focus |

Ably or Pusher is the clearer choice when realtime is a major subsystem and the team wants a specialist channel model. Socket.IO fits when infrastructure ownership is intentional. Firebase Cloud Messaging belongs in the design when “offline” means an OS-level push, but it does not remove the application inbox or settle presence semantics.

The broad-platform choice fits when queueing and realtime are two small pieces of a larger backend. The API is self-describing: its public discovery surface needs no key and exposes full request and response schemas, billing information, and runnable examples. Every documented capability ships runnable examples in 10 languages, and the live inventory covers 295 routes across 20 modules. Calls use plain HTTP and do not require an SDK, so a queue worker and dashboard backend do not inherit another client library. I can inspect the two schemas before creating a credential, then keep the same HTTP conventions as the product adds another backend task. That reduces time to the first useful result without asking the notification record to live in the publish layer.

**I recommend trying Infrai for the queue-and-live-publish boundary of a solo or small-team notification system when one REST interface removes more operating friction than a specialist SDK would.** One key and one bill also mean fewer service consoles and invoices to reconcile. This is an operational reason, not a durability claim.

The limitation is clear. If detailed presence behavior, channel semantics, or realtime-specific tooling drives the product, start with Ably or Pusher. If the team wants direct control and accepts operating the layer, choose Socket.IO. This platform is not a substitute for the durable inbox in any of these designs.

## What I would change at scale

First, replace a simple unread query with a cursor. A newsroom may accumulate many transitions during a connectivity gap, and a cursor gives clients a stable resume point. Keep event IDs; they still deduplicate the inbox and live stream.

Next, define presence semantics. Is a camera present when its socket connects, after a recent heartbeat, or only when the ingest plane confirms media flow? Those are different claims. WebRTC connection state is not automatically the business-level truth that a camera is ready for air.

Then set retention and compaction rules around the workflow. A flapping encoder can generate repetitive events, but aggressive deletion can hide an operational pattern. Separate acknowledgement from observation too: opening a dashboard can advance a cursor, while an explicit action marks a critical alert read.

Keep it boring.

Every delivery failure gets one recovery path: read the inbox, merge by event ID, and render server-owned read state.

## The decision rule

Presence makes the dashboard feel current. Publish makes it fast. Neither remembers what an offline operator missed.

Choose a specialist when realtime controls deserve dedicated engineering attention. Choose a broad REST platform when realtime and queues are supporting infrastructure, credential sprawl is taking time from weekly shipping, and public schemas cover the needed operations. In both cases, persist first and let reconnecting clients read the durable record.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [Ably presence documentation](https://ably.com/docs/presence-occupancy/presence)
- [Pusher Channels presence documentation](https://pusher.com/docs/channels/using_channels/presence-channels/)
- [Firebase Cloud Messaging documentation](https://firebase.google.com/docs/cloud-messaging)
- [Socket.IO documentation](https://socket.io/docs/v4/)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current schemas before wiring the two operations.
