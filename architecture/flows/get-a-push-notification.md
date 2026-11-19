# Flow: get a push notification

> **In one minute:** A browser that allowed notifications gives the web app a push subscription, and the web
> app saves it in chatty-chat. Every saved channel event is also a row in `channel_events`; the outbox relay
> sends those rows to Kafka in order. The push worker reads `message.created` from Kafka, picks the
> channel members who want a notification and have no Chatty tab on screen, and sends each of their
> browsers a Web Push. The browser shows it even when Chatty is closed.

## The picture

```mermaid
sequenceDiagram
    autonumber
    actor U as Bob
    participant W as Bob's browser
    participant H as chatty-chat
    participant DB as chat_db
    participant RL as Outbox relay
    participant K as Kafka
    participant PW as Push worker
    participant C as chatty-core
    participant R as Redis
    participant P as Push service

    U->>W: Settings: Notifications, Turn on
    W->>W: Allow, pushManager.subscribe(VAPID public key)
    W->>H: POST /api/chat/v1/push/subscriptions {endpoint, p256dh, auth}
    H->>DB: upsert by endpoint, keep my 10 newest
    H-->>W: 204

    Note over H,P: Ana sends a message in a channel Bob is in
    H->>DB: COMMIT message + channel_events row (published_at null)
    DB-->>RL: NOTIFY chat_events
    RL->>DB: lock the oldest unpublished rows
    RL->>K: send each, key = channel_id, wait for the ack
    RL->>DB: published_at = now(), COMMIT
    K-->>PW: message.created
    PW->>C: GET /internal/channels/{channel_id} (cached)
    PW->>PW: Members by notify_level and muted
    PW->>R: Bob has a tab on screen? (visible:{bob})
    PW->>R: SET push:{message_id}:{bob} NX (first time only)
    PW->>DB: Bob's subscriptions
    PW->>P: Web Push to each browser (VAPID signed, encrypted)
    P-->>W: the notification
    PW->>K: commit the offset
```

## Step by step

**1. Turn it on** (the web app, Settings)

1. `NotificationsSection` asks the browser for permission (from a click) and registers the service worker
   `/sw.js?user={user_id}`.
2. `pushManager.subscribe` with the VAPID public key from `VITE_VAPID_PUBLIC_KEY` gives an `endpoint`,
   `p256dh` and `auth`.
3. `POST /push/subscriptions` saves them (D-13). The endpoint must be `https` on a known push service
   (Google, Mozilla, Apple, Windows), so the worker can't be pointed at an internal address. A known
   endpoint gets the new keys and the current user, so a shared computer only gets pushes for whoever is
   logged in. Each user keeps their 10 newest.
4. Turning it off, and logging out, unsubscribe the browser and call `DELETE /push/subscriptions`, which
   only deletes the caller's own row.

**2. The outbox relay** (`python -m app.jobs.relay`, its own container)

1. Every channel event is written to `channel_events` in the same transaction as the change, with
   `published_at` null. A trigger sends `NOTIFY chat_events` on commit, which wakes the relay; it also
   polls every 0.2 s, in case a `NOTIFY` was lost.
2. The relay locks up to 100 unpublished rows, oldest `id` first, sends each to the Kafka topic
   `chat.events` with `key = channel_id` and waits for the acknowledgement, then sets `published_at`.
   One channel's events land in one partition, in order.
3. Only one relay runs at a time: it holds a Postgres advisory lock, and a second one waits for it. Two
   relays could reorder a channel's events.

**3. The push worker** (`python -m app.jobs.push_worker`, its own container)

1. It reads `chat.events` in the consumer group `push-worker`, one event at a time, and commits the
   offset after each. Only `message.created` counts, and only if it is less than 10 minutes old.
2. The channel comes from the same cache as the API, or chatty-core. An archived channel or a deleted
   team gets nothing.
3. Who gets it: every member but the author who isn't muted, with `notify_level` `all`, or `mentions` and
   mentioned.
4. Minus anyone with a tab on screen in the last 45 s: the heartbeat's `visible` keeps
   `visible:{user_id}` in Redis. A Redis error counts as not looking: an extra notification is better
   than a missed one.
5. `SET push:{message_id}:{user_id} NX` (1 day) makes sure a user gets each message once, even if Kafka
   delivers it again after a crash.
6. Each browser of the user gets `{type, team_id, channel_id, message_id, seq, mentioned, preview}`,
   with the first 100 characters of the body, signed with the VAPID private key and encrypted for that
   browser. A push service that answers `404` or `410` means the browser unsubscribed: the row is deleted.

**4. The browser.** `sw.js` shows "New message" or "You were mentioned" with the preview, mentions shown
as `@you` or `@someone`, one notification per channel. A click opens the channel in a Chatty tab, or a
new one.

## If a step fails

| Step | What happens |
|---|---|
| Save: not a known push service, or keys of the wrong length | `400 validation_error` |
| Kafka down | The relay rolls back and retries every second; rows stay unpublished and are sent when it is back. The API and live events over Redis keep working |
| The relay crashes after sending, before marking | Those events are sent again. Kafka delivers at least once; the dedupe key stops a second push |
| chatty-core, the database or Redis down in the worker | It retries the same event every second, so nothing is lost. After 10 minutes the event is too old and is skipped |
| One browser's push service fails | Logged, the others still get theirs; it isn't retried |
| `VAPID_PRIVATE_KEY` not set | The worker doesn't start. The web app hides the button if `VITE_VAPID_PUBLIC_KEY` is empty |

## ⚠️ Known gaps

- **A deactivated user still gets pushes.** Deactivation keeps their channel memberships and nothing
  deletes their subscriptions.
- **One event at a time.** Up to 6 workers (one per partition) can share the work if it lags.
- **A worker crash between the dedupe key and the send** loses that user's push for that message.
- **The relay doesn't notice it lost its advisory lock** if its control connection drops, so a second
  relay could start next to it.

## Key code

| File | What is there |
|---|---|
| `chatty-chat/app/api/routers/push.py` | `POST` and `DELETE /push/subscriptions` |
| `chatty-chat/app/schemas/push.py` | The endpoint and key checks, the push payload |
| `chatty-chat/app/services/push_subscription_service.py` | Save (upsert, 10 newest), delete |
| `chatty-chat/app/jobs/relay.py` | The relay loop: advisory lock, `LISTEN`, poll |
| `chatty-chat/app/services/outbox_relay.py` | `publish_batch`: lock, send, mark |
| `chatty-chat/app/jobs/push_worker.py` | The consumer loop and its retries |
| `chatty-chat/app/services/push_service.py` | `push_recipients`, `PushService.handle` |
| `chatty-chat/app/clients/web_push.py` | `WebPushClient`, `PushDedupe` |
| `chatty-chat/app/clients/kafka.py` | The producer and consumer, keyed by `channel_id` |
| `chatty-web/src/features/push/` | Turn on and off, the settings section |
| `chatty-web/public/sw.js` | The service worker: show the notification, open the channel |
