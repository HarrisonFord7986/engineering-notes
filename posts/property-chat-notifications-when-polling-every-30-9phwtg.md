# Property Chat Notifications — When Polling Every 30 Seconds Needs Realtime

Use 30-second polling for notification badges and inbox summaries. Use a persistent realtime channel for messages that must appear while a tenant and property manager are actively chatting. In either case, authorize each fetch or connection against the room and user; transport speed never compensates for an over-broad token.

| Choice | Delivery expectation | Reconnect work | Trust boundary | Best fit |
| --- | --- | --- | --- | --- |
| Poll every 30 seconds | New items may wait for the next poll | Ordinary request retry | Server checks every request | Badge counts, maintenance updates, non-urgent inboxes |
| Persistent channel | New items can arrive on an open connection | Resume, deduplicate, and catch up | Server checks connection and channel subscription | An open tenant-manager chat |
| Hybrid | Live while connected, fetch after gaps | Cursor-based catch-up | Same room policy on both paths | Chat plus durable notification history |

**Recommendation:** choose the hybrid for property chat. Keep the database-backed inbox as the record, send a lightweight wake-up event on the live channel, and recover by cursor after reconnect. This spends complexity where tenants can see it. A 30-second badge delay is usually tolerable; a message that vanishes during an elevator dead zone is not.

That distinction matters more than fashionable transport labels. For a one-person SaaS, every hour spent on connection state is an hour not spent shipping the lease workflow. Ship weekly. Outsource undifferentiated transport if that buys back time, but retain authorization and recovery semantics in application code because those are part of the product's correctness.

## Should notifications use realtime or polling every 30 seconds?

The useful question is not “realtime or polling?” It is: what breaks during the maximum expected delay?

Polling has limits.

Suppose a tenant reports water entering unit 4B. A property manager opens the room, asks whether the shutoff valve is accessible, and waits. In that active conversation, a response delayed by up to one polling interval can create duplicate calls or conflicting instructions. The room deserves a persistent channel.

Now consider “your monthly statement is ready.” The notification is durable, has no conversational turn-taking, and remains useful after several minutes. Thirty-second polling is enough. It is also easier to inspect with ordinary request logs and easier to retry after a laptop wakes.

Set the expectation before picking a protocol. Define an observable delivery target for active-room messages, a looser target for the inbox, and a recovery target after disconnection. Avoid claiming “instant.” Networks pause, browsers sleep, and processes restart. The system should promise what it can verify: durable acceptance, ordered retrieval within a room, and eventual catch-up from a known cursor. Write those targets into tests before adding a live connection, because otherwise a green “connected” indicator can hide the failure that matters: a stored message never appearing after a gap. The trade-off is concrete. Tighter delivery and recovery targets demand more state, more observability, and more time spent testing failure boundaries.

This is the revenue-per-hour test. A faster badge rarely changes retention. A trustworthy maintenance conversation can.

## Scope tokens to rooms, not socket lifetimes

A long-lived connection changes how often authorization code runs, not what it must prove. The server still needs an authenticated principal, the requested property or room, and the allowed operation. A token that merely says `user:123` leaves too much interpretation to the client. The client is not the policy engine.

Use short-lived access tokens whose audience identifies the notification service. Resolve room membership on the server. If a tenant moves out or a contractor's assignment ends, the next subscription check must reject that room even when the underlying connection remains open. Revalidate on subscription changes and at a bounded interval appropriate to the revocation requirement; do not treat successful connection setup as permanent permission.

**One connection may carry many rooms, but one authorization decision must not.** Keep routing and permission separate. A client-supplied room ID is a request, never evidence.

The same rule applies to polling. A query such as `after=8041` controls where retrieval starts; it must not expand which rows the caller may read. Filter by the server-derived principal and room membership first, then apply the cursor. Return opaque cursors if exposing database identifiers would leak tenancy or volume information.

WebRTC is usually the wrong primitive for this notification path. Its standard centers peer connections and media or data exchange between endpoints, including ICE negotiation. A tenant notification inbox needs a durable server record and server-enforced room access. Peer connectivity does not remove either requirement.

## Make reconnect a data problem

The live event should wake the client, not become the sole copy of the message. Persist first. Then publish an event carrying a stable message ID and room-local cursor. On reconnect, the client asks for everything after its last committed cursor. Duplicate delivery becomes harmless when the reducer keys by message ID.

Here is the application boundary I would keep even if the transport provider changes:

```ts
type RoomEvent = {
  id: string;
  roomId: string;
  cursor: string;
  kind: "message.created" | "message.updated";
  occurredAt: string;
};

type Session = {
  userId: string;
  audience: "property-notifications";
};

interface RoomPolicy {
  canRead(userId: string, roomId: string): Promise<boolean>;
}

interface EventStore {
  after(roomId: string, cursor: string | null): Promise<RoomEvent[]>;
}

async function catchUpRoom(
  session: Session,
  roomId: string,
  cursor: string | null,
  policy: RoomPolicy,
  store: EventStore,
): Promise<RoomEvent[]> {
  if (session.audience !== "property-notifications") {
    throw new Error("invalid token audience");
  }

  if (!(await policy.canRead(session.userId, roomId))) {
    throw new Error("room access denied");
  }

  return store.after(roomId, cursor);
}
```

Run that authorization path for HTTP catch-up and live subscription. Two transports, one policy. That is less exciting than a socket demo and far more valuable when a lease changes at noon.

Client state needs three markers: the highest cursor rendered, the highest cursor durably acknowledged locally, and connection status. Advance the cursor only after the event is accepted into the local store. If rendering fails, reconnecting from an earlier cursor produces a duplicate rather than a hole. Good trade.

Test gaps explicitly. Disconnect after the server persists an event but before the client receives it. Deliver the same event twice. Revoke room access while a connection is open. Resume with an expired token. Send events for two rooms with independent cursors. These tests do not need timing sleeps; drive persistence, publication, authorization, and reconnection as separate steps.

Operations should expose the same boundaries. Track accepted-to-published delay, published-to-acknowledged delay where acknowledgements exist, catch-up batch size, duplicate rate, authorization denials, and reconnect frequency. Correlate logs with event ID and room ID, while keeping message bodies and access tokens out of logs.

## When is the polling runner-up better?

Polling wins when the notification is durable, low urgency, and cheap to query incrementally. It also wins early in a product's life when traffic is modest and the team cannot yet operate connection fan-out, resume behavior, and revocation confidently. A conditional request can avoid transferring an unchanged representation, although the server still has to receive and evaluate the request.

The hybrid has a real limitation: it makes the client reconcile a live stream with catch-up results. It is a poor fit when every notification can wait 30 seconds, because there is then no user consequence large enough to justify connection lifecycle code, duplicate handling, and live-channel monitoring. Plain polling is the better choice in that case. At the other extreme, polling is a poor fit for active conversational turns because its delay is built into the interaction rather than caused by an occasional failure.

Use polling for the global inbox even if rooms are live. Pause or slow it when the page is hidden, trigger an immediate refresh when the page becomes visible, and add random jitter so every browser does not request on the same boundary. On error, use bounded exponential backoff. Keep a manual refresh path.

There are clear cases where 30 seconds is not fine: an operator is waiting in the open room, the notification changes an immediate safety action, or the interface implies synchronous turn-taking. There are equally clear cases where realtime is waste: a document-ready notice, a weekly owner summary, or an unread count that the user sees only after navigating back to the inbox.

Do not split the data model. Both delivery paths should read the same durable events and apply the same access policy. Then moving one notification class from polling to a live channel is a transport change, not a migration of truth.

Keep one truth.

## The decision rule

Choose from the user's consequence, then price the engineering hours. Active property chat gets a persistent channel plus cursor recovery. Durable inbox notifications get 30-second polling. Both get room-scoped server authorization, an audience-bound token, deduplication, and a database record that outlives the connection.

Fast delivery is visible. Correct recovery earns trust.

## References

- W3C, “WebRTC: Real-Time Communication in Browsers”: https://www.w3.org/TR/webrtc/
- IETF RFC 6455, “The WebSocket Protocol”: https://www.rfc-editor.org/rfc/rfc6455
- WHATWG HTML Living Standard, “Server-sent events”: https://html.spec.whatwg.org/multipage/server-sent-events.html
- IETF RFC 9110, “HTTP Semantics” (conditional requests): https://www.rfc-editor.org/rfc/rfc9110
- IETF RFC 9700, “Best Current Practice for OAuth 2.0 Security”: https://www.rfc-editor.org/rfc/rfc9700
