# Live events: sockets, Redis, gateways and the cache

How a change reaches every open browser tab, and how chatty-chat keeps chatty-core's answers in a
cache without serving stale access.

> **In one minute:** A browser keeps one WebSocket open to chatty-chat. Writes never go over it: they
> go through REST, and after the change is saved chatty-chat **publishes** an event on Redis pub/sub.
> Every chatty-chat process runs one **gateway**: a single Redis subscription that receives events and
> forwards them to the sockets connected to that process. A gateway only subscribes to the channels its
> own sockets are in. chatty-core publishes its own events (`core.events`) the same way, and chatty-chat
> uses them to update open sockets and to delete stale cache keys. Pub/sub keeps nothing: a missed event
> is recovered from the database with `POST /sync`.

## The pieces

| Piece | What it is | Code (chatty-chat) |
|---|---|---|
| WebSocket | One long-lived connection per browser tab, so the server can send at any time | `app/api/routers/ws.py` |
| Process | One running uvicorn with the FastAPI app. There can be several; each is a gateway for its own sockets | `app/main.py` |
| Gateway | One per process. Owns one Redis pub/sub connection, receives events, sends them to local sockets | `app/services/gateway.py` |
| ConnectionRegistry | In memory, per process: which sockets each user has, and which sockets are in each channel and team | `app/services/connection_registry.py` |
| EventBus | The sending side: one `PUBLISH` on `chat.events:{channel_id}` per channel event, on `chat.users:{user_id}` for a live message to one user, and on `chat.typing:{channel_id}` when someone types | `app/services/event_bus.py` |
| PresenceStore | Who is online, in Redis, shared by every process | `app/clients/presence.py` |
| Redis cache | chatty-core's `/internal` answers, kept 5 minutes, with version counters | `app/clients/core.py` |
| `core.events` listener | Deletes the cache keys a chatty-core event makes stale | `app/clients/core_events.py` |
| `channel_events` table | Every channel event with its `seq`, the source of truth for catching up | `app/models/channel_event.py` |

```mermaid
flowchart LR
    T1[Browser tab] <-->|WebSocket| G1
    T2[Browser tab] -->|REST POST /messages| S

    subgraph PA[chatty-chat process A]
        G1[Gateway] --- R1[ConnectionRegistry]
        L1[core.events listener]
    end

    subgraph PB[chatty-chat process B]
        S[MessageService] --> EB[EventBus]
        G2[Gateway]
    end

    S -->|commit| DB[(chat_db)]
    EB -->|PUBLISH chat.events:42aa| R[(Redis)]
    R -->|chat.events:42aa| G1
    C[chatty-core] -->|PUBLISH core.events| R
    R -->|core.events| G1
    R -->|core.events| L1
    R -.->|only if B has sockets in 42aa| G2
```

The process that saves a message is usually not the one holding the receivers' sockets. Redis
pub/sub connects them: every process publishes to Redis and listens to Redis, so it doesn't matter which
process a browser is connected to.

## Redis: three jobs

Redis does three unrelated things for chatty-chat.

| Job | How it works | Keys or channels |
|---|---|---|
| **Cache** | Plain keys with a TTL. A read that finds the key is a **hit**; a read that doesn't is a **miss**, and chatty-chat asks chatty-core and stores the answer | `members:{user_id}`, `channel:{channel_id}`, and the counters `version:members:{user_id}`, `version:channel:{channel_id}`, `version:all` |
| **Pub/sub** | Named channels. A `PUBLISH` reaches every connection that is `SUBSCRIBE`d to that name **right now**. Nothing is stored: a subscriber that isn't listening misses the message | `chat.events:{channel_id}` and `chat.typing:{channel_id}` (one each per chat channel), `chat.presence:{team_id}` (one per team), `chat.users:{user_id}` (one per user with an open socket), `core.events` (chatty-core's) |
| **Presence** | One line per open socket, and a set of online users per team. It isn't a copy of anything: this is the only record. See [see who is online](flows/see-who-is-online.md) | `presence:{user_id}`, `online:{team_id}` |

chatty-chat uses two Redis clients (`app/main.py`):

- `app.state.redis`: cache reads and writes, and `EventBus.publish`. 2 s socket timeouts.
- `events_redis`: the listeners. Retries are off, so a dropped connection surfaces at once and the code's
  own reconnect loops take over. Each `pubsub()` opens its own connection: a connection in subscribe mode
  can't run `GET` or `PUBLISH`. `pubsub.listen()` waits with no timeout, but redis-py 8.1 turns on TCP
  keepalive by default, so a connection that dies silently fails after about 45 s and the loops reconnect.

## Start-up

The app's lifespan creates the registry and the gateway and starts two background tasks:

| Task | Subscribes to | Job |
|---|---|---|
| `gateway.run()` | `core.events`, `chat.events:{channel_id}` and `chat.typing:{channel_id}` for every channel a local socket is in, `chat.presence:{team_id}` for every team a local socket is in, and `chat.users:{user_id}` for every user with a local socket | Keep open sockets right |
| `listen_core_events()` | `core.events` | Keep the cache right |

Both tasks get every `core.events` message, because pub/sub gives each subscriber its own copy. A crash
in either task is logged (`log_crash`), and `/health` then answers `503` with `"tasks": "error"`, because
that process no longer keeps its sockets or its cache right (CHAT-269). On shutdown both are cancelled, and uvicorn closes the open sockets
with `1012`.

## Connecting a socket

```mermaid
sequenceDiagram
    autonumber
    participant B as Browser
    participant WS as ws.py
    participant CC as CoreClient
    participant C as chatty-core
    participant GW as Gateway
    participant R as Redis

    B->>WS: GET /api/chat/v1/ws?token=JWT
    WS->>B: accept()
    WS->>WS: check the token (RS256, chatty-core's public key)
    WS->>CC: get_memberships(user_id)
    CC->>R: MGET members:{user_id} + its versions
    opt cache miss
        CC->>C: GET /internal/users/{user_id}/memberships
        C-->>CC: teams, channels, is_active
        CC->>R: cache it, only if no event came in meanwhile
    end
    WS->>GW: connect(user, socket, channel_ids, team_ids)
    GW->>R: SUBSCRIBE chat.users:{user}, chat.events:{channel}, chat.typing:{channel} and chat.presence:{team} without a local socket yet
    GW->>R: Add the socket's line to presence:{user_id}, the first one sends online
    loop until a close
        B->>WS: heartbeat / watch / unwatch
    end
    WS->>GW: disconnect() (always, in finally)
    GW->>R: UNSUBSCRIBE the user, channels and teams that lost their last local socket
    GW->>R: Remove the socket's line, the last one sends offline
```

1. chatty-chat **accepts first**, then checks. A socket closed before `accept()` reaches the browser as
   `1006` with no reason; after it, the browser gets our close code.
2. The token comes in the URL (browsers can't set headers on a WebSocket). Bad or expired: `4001`.
   uvicorn logs the URL, so `setup_logging` replaces `token=…` with `token=***` in its log lines.
3. The user's memberships come from the cache or chatty-core. Unknown user: `4001`. Deactivated: `4003`.
   chatty-core unreachable: `1011 core_unavailable`.
4. `gateway.connect` adds the socket to the registry under the user and under each of their channels and
   teams, and records it in Redis, which tells if the user just came [online](flows/see-who-is-online.md).
5. The socket then waits for client messages until a deadline passes or it closes.

## The ConnectionRegistry

Everything here is in memory, for this process only. The sockets of each user are one map; channels and
teams are each a `SocketGroups`, the same class created twice:

```mermaid
flowchart LR
    U7[user u7] --> T1[tab 1]
    U7 --> T2[tab 2]
    U3[user u3] --> T3[u3's tab]
    T1 --> C8["#c8"]
    T2 --> C42["#42aa"]
    T3 --> C42
```

| Map | Answers | Used for |
|---|---|---|
| user → sockets | Which sockets does this user have? | `user.deactivated`, `membership.changed`, `unread` |
| channel → sockets | Who gets an event for this channel? | Every channel event |
| socket → channels | Which channels is this socket in? | A quick `leave_all` on disconnect |
| team → sockets | Who gets a presence frame for this team? | Every `presence` frame |
| socket → teams | Which teams is this socket in? | Which teams get `offline`, and a quick `leave_all` |

`join()` returns `True` when a channel or team gets its **first** local socket, and `leave()` /
`leave_all()` report when one loses its **last**. That is the signal to `SUBSCRIBE` or `UNSUBSCRIBE` in
Redis. So a process subscribes to each channel and team once, whether 1 or 500 of its sockets are in it,
and only to the ones its own users need. `add()` and `remove()` do the same for a user: the user's first
local socket subscribes to `chat.users:{user_id}`, and the last one unsubscribes.

## From a saved change to every tab

```mermaid
sequenceDiagram
    autonumber
    participant API as POST /messages (any process)
    participant DB as chat_db
    participant EB as EventBus
    participant R as Redis
    participant GA as Gateway A (has sockets in #42aa)
    participant GC as Gateway C (no sockets in #42aa)
    participant S as Sockets in #42aa

    API->>DB: one transaction: next seq, message, channel_events row
    DB-->>API: COMMIT
    API->>EB: publish(42aa, envelope)
    EB->>R: PUBLISH chat.events:42aa
    R-->>GA: the event
    Note over GC: not subscribed, gets nothing
    GA->>S: {"type": "event", "event": …} to each socket, 5 s limit each
```

1. The event is published **only after the commit**, so nobody hears about a change that rolled back.
2. Every process with a socket in the channel is subscribed to `chat.events:{channel_id}` and gets the
   event. The gateway looks up the channel's local sockets and sends them all at once.
3. A socket that doesn't take the frame within 5 s is removed from its channels and closed with
   `1001 too_slow` in the background, so its browser reconnects and syncs. The socket's own handler then
   sees it is already closed and doesn't close it a second time. It stays in its teams until then, because
   `disconnect` needs them to send `offline`, but it gets no more `presence` frames.
4. The sender's own tab gets the message twice, in the REST answer and as an event. The web app keeps one,
   by `id`.

**The live path may lose an event, and that is fine.** `EventBus.publish` logs a Redis error and the REST
call still succeeds. The event is in `channel_events` with its `seq`, so a client that sees a gap or
reconnects gets it from [`POST /sync`](flows/reconnect-and-sync.md).

## A read reaches the user's other tabs

```mermaid
sequenceDiagram
    autonumber
    participant L as Laptop tab
    participant API as POST /channels/{id}/read (any process)
    participant DB as chat_db
    participant R as Redis
    participant GA as Gateway A (has a socket of the user)
    participant P as Phone

    L->>API: {seq}
    API->>DB: one transaction: lower seq to last_seq, move read_state forward, count unread
    DB-->>API: COMMIT
    API->>R: PUBLISH chat.users:{user} unread
    R-->>GA: the message
    GA->>P: {"type": "unread", "channel_id", "unread", "mentions"}
    API-->>L: 200 {channel_id, last_read_seq, unread, mentions}
```

1. A read position is private, so it can't go on `chat.events:{channel_id}`, which every member hears.
   Each user has their own channel instead, `chat.users:{user_id}` (D-12).
2. The message is published **only after the commit**, like channel events.
3. The tab that sent the request gets it too: a REST request can't tell which socket is its own. It
   carries the same counts as the answer, so nothing changes.
4. `unread` has no `seq` and isn't stored. If a socket misses it, the web app's `GET /unread` after the
   reconnect brings the badges back in line.

## Someone is typing

While someone types, their tab sends a small `typing` message on the WebSocket, at most every 3 s. chatty-chat
checks it and passes it to the others in the channel. It's a short-lived signal, not data: no database row, no
`seq`, no event row, no REST endpoint, no `/sync` and no Kafka. If one is lost nothing breaks, because the next
one comes 3 s later or the text just goes away.

```mermaid
sequenceDiagram
    autonumber
    participant A as Ana's tab
    participant GA as Gateway A
    participant R as Redis
    participant GB as Gateway B
    participant B as Bob's tab

    A->>GA: {"type": "typing", "channel_id": "42aa…"}
    GA->>GA: Run the checks, add Ana's user_id from the socket
    GA->>R: PUBLISH chat.typing:42aa
    R-->>GA: the frame
    R-->>GB: the frame
    GA->>GA: Skip Ana's own tabs and stuck sockets
    GB-->>B: {"type": "typing", "channel_id": "42aa…", "user_id": "ana"}
    Note over A,B: Ana keeps typing: step 1 repeats every 3 s
    Note over A,B: Ana stops: no more frames, nothing else is sent
```

**The checks.** Each one that fails ignores the message: no `error`, and the socket stays open.

```mermaid
flowchart TD
    M[typing arrives] --> C1{Socket is in the channel, and not only watching it?}
    C1 -->|no| X[Ignore it]
    C1 -->|yes| C2{Less than 3 s since this socket's last one for this channel?}
    C2 -->|yes| X
    C2 -->|no| S[Save the time]
    S --> C3{get_writable_channel: can the user still write?}
    C3 -->|no, or an error| X
    C3 -->|yes| P[PUBLISH on chat.typing:channel_id]
```

1. **In the channel**, from the registry, in memory. A socket that only watches the channel is in its sockets
   too, but it can't write.
2. **The 3 s limit**, in memory: `Gateway._typing_at` keeps the last time per socket and channel, and
   `disconnect` removes the socket. It's per socket, so two tabs can send two frames per 3 s.
3. **Can write**, which refuses archived channels. It's normally one cached Redis read. It comes last, so random
   channel IDs can't cause chatty-core calls.
4. **Publish**, with `EventBus.send_typing`. A Redis error is logged, and the socket stays open.

`typing` doesn't reset the 60 s heartbeat timer. Only a heartbeat keeps the socket alive.

**Who it is.** The client never says. The token is checked when the socket opens, and the server adds `user_id`
from that user, so nobody can pretend someone else is typing. A `user_id` sent by the client is ignored.

**Who hears it.** Whoever gets a channel's events also gets its typing: `_subscribe_channels` listens to
`chat.events:{channel_id}` and `chat.typing:{channel_id}` together, so the channel's first local socket
subscribes to both and the last one unsubscribes from both. `handle_typing` sends the frame to the channel's local
sockets, watchers included, except the typing user's own tabs and stuck sockets.

**Stopping.** There is no "stopped typing" message, so a closed tab or a crash needs no cleanup. The frames stop,
and the web app hides the text after 5 s, or as soon as that user's `message.created` arrives.

## chatty-core's events on open sockets

The gateway's own `core.events` subscription keeps sockets that are already open right:

| Event | What the gateway does |
|---|---|
| `membership.changed` `added` | The user's sockets join the channel (`SUBSCRIBE` if first). A watch on that channel becomes a membership |
| `membership.changed` `removed` | The user's sockets leave the channel (`UNSUBSCRIBE` if last). Its events stop right away |
| `channel.updated` | Sends `{"type": "channel.updated", …}` to the channel's sockets, so the sidebar refreshes |
| `user.deactivated` | Closes the user's sockets with `4003` |
| `team.member.changed` `joined` | The user's sockets join the team (`SUBSCRIBE` if first), and `online` is sent to it |
| `team.member.changed` `left` or `removed` | Ends a watch on that team's channel, and the sockets leave the team (`UNSUBSCRIBE` if last), which gets `offline`. The channel memberships come as their own `membership.changed removed` |
| `team.deleted` | Nothing: the team's channels reject writes, so no new events arrive |

The gateway handles `core.events`, the `chat.events:*`, `chat.typing:*`, `chat.presence:*` and `chat.users:*` channels on **one** connection, one message at a
time, so a `membership.changed removed` that arrived first is applied before the next channel event is
forwarded. Every process gets the same event and updates its own sockets.

## Watching a public channel

A socket can follow one public channel the user hasn't joined, for example while they look at it.

```mermaid
stateDiagram-v2
    [*] --> NotWatching
    NotWatching --> Watching: watch(X), X is public in one of my teams
    NotWatching --> NotWatching: watch(X), not public in my teams, error cannot_watch
    Watching --> Watching: watch(Y) replaces X
    Watching --> NotWatching: unwatch(X), or I leave X's team
    Watching --> Member: membership.changed added for X
    Member --> [*]
```

- At most one watched channel per socket. If the user is already a member, nothing changes, so a later
  `unwatch` can't remove a membership.
- A watch uses the same `join()` / `SUBSCRIBE` as a membership.

## Keeping the socket alive

| Mechanism | Who | Catches |
|---|---|---|
| Ping/pong | uvicorn pings every 20 s; the browser answers on its own. No pong within 20 s closes the socket (`1011`) | A dead connection: a sleeping laptop, a dropped network |
| Heartbeat | The web app sends `{"type": "heartbeat", "visible": …}` every 20 s. None for 60 s closes with `4002` | A frozen app; `visible` says if the tab is on screen |
| Token deadline | The same timer watches the token's `exp` and closes with `4001` | An expired token; the web app should refresh and reconnect about a minute earlier |

Only a valid heartbeat resets the 60 s timer. Invalid JSON, unknown types and binary frames are ignored.
A message over 4 KB is closed with `1009` by uvicorn (`--ws-max-size 4096`).

## Close codes

| Code | Reason | Sent by |
|---|---|---|
| `4001` | `token_expired`: bad or expired token, unknown user | `ws.py` |
| `4002` | `heartbeat_timeout` | `ws.py` |
| `4003` | `user_inactive` | `ws.py`, gateway on `user.deactivated` |
| `1001` | `gateway_restart` (Redis came back) or `too_slow` | gateway |
| `1009` | a message over 4 KB | uvicorn |
| `1011` | `core_unavailable` at connect, or no pong | `ws.py`, uvicorn |
| `1012` | shutdown | uvicorn |

After any close but `4003`, the web app reconnects with backoff and calls `POST /sync`
([reconnect and sync](flows/reconnect-and-sync.md)). The full table is in
[events.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/events.md#ws-close).

## The cache and its versions

`CoreClient` keeps chatty-core's `/internal` answers so not every request calls chatty-core:

1. **Read:** one `MGET` gets the key and its two version counters (the key's own and `version:all`).
2. **Hit:** use the cached body.
3. **Miss:** ask chatty-core (2 s timeout), then store the answer with a Lua script that writes it **only
   if neither counter changed** since step 1.
4. **An event** (`core_events.py`) increments the key's counter, then deletes the key, in one
   transaction. `team.deleted` and the full clear after a reconnect increment `version:all`.

```mermaid
sequenceDiagram
    autonumber
    participant Q as Request (cache miss)
    participant C as chatty-core
    participant L as core.events listener
    participant R as Redis

    Q->>R: MGET members:u7, version:members:u7, version:all
    R-->>Q: miss, version 4, all 2
    Q->>C: GET memberships
    C-->>Q: the old answer (the change isn't committed yet)
    C->>L: membership.changed, after its commit
    L->>R: MULTI: INCR version:members:u7 (5), DEL members:u7, EXEC
    Q->>R: Lua: SET only if versions are still 4 and 2
    R-->>Q: refused
    Note over R: the next request misses and gets the new answer
```

**Why the versions:** without them, a request that got its answer just before the change could write it
back just after the event deleted the key, and the removed user would keep access until the TTL ends
(5 minutes). Increment first, then delete: an old answer written before is deleted, one written after is
refused. A script runs as one step in Redis, so nothing can come in between the check and the write.

- If Redis is down, the cache is skipped: every request asks chatty-core and nothing is cached.
- On every (re)subscribe, the listener bumps `version:all` and deletes all `members:*` and `channel:*`
  keys, because events sent while it wasn't listening are lost (D-08). It scans and deletes in batches of
  500, never `KEYS`.
- The counters expire after a day. A request lasts about 2 s, so none can see its counter expire mid-way.

## If something is down

| Down | What happens |
|---|---|
| Redis, while publishing | `EventBus.publish` logs it; the REST call succeeds; clients get the event from `POST /sync` |
| Redis, while listening | The gateway retries (1 s, doubling up to 30 s). When it is back it closes every local socket with `1001 gateway_restart`, because pub/sub doesn't replay what was missed, and subscribes again |
| Redis, for the cache | Every access check asks chatty-core; nothing is cached |
| Redis, for typing | Logged; sockets stay open. Nobody sees typing, and messages aren't affected |
| Redis, for presence | Logged; sockets stay open. Changes meanwhile aren't heard, and `GET /teams/{team_id}/presence` answers an empty list |
| A socket stops reading | 5 s send limit, then it leaves its channels and is closed with `1001 too_slow` |
| chatty-core, at connect | `1011 core_unavailable` |
| chatty-core, later | Access checks that aren't cached fail with `503 core_unavailable` |
| The process shuts down | uvicorn closes its sockets with `1012`; the browsers reconnect to another process |

## ⚠️ Known gaps

- **Watch uses the memberships loaded at connect.** Someone removed from a team can start a new watch on
  its public channels until they reconnect ([CHAT-231](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-231)).
- **One stuck socket delays its gateway's events** by up to 5 s once, before it is dropped. A queue per
  socket is the fix if the load test shows it ([CHAT-224](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-224)).
- **The web app doesn't send or show typing yet** ([CHAT-217](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-217)).
  The heartbeat's `visible` is stored but not used yet.
- **Presence has a few gaps of its own:** a crash sends no `offline`, a page refresh can flicker, and a close
  and a connect at the same moment can end in the wrong order. See
  [see who is online](flows/see-who-is-online.md).
- **Kafka isn't used yet.** Events go over Redis pub/sub; the outbox (`channel_events.published_at`) has no
  relay yet. See the [data map](data-map.md#kafka).

Full list: [known limitations](known-limitations.md).

## Key code

| File | What is there |
|---|---|
| `chatty-chat/app/main.py` | The lifespan: both Redis clients, the gateway and the `core.events` listener |
| `chatty-chat/app/api/routers/ws.py` | Accept, token and memberships check, client messages, `handle_typing`, deadlines, close codes |
| `chatty-chat/app/services/gateway.py` | `Gateway`: `connect`, `disconnect`, `heartbeat`, `can_type`, `watch`, `run`, channel, typing, presence and core events, the 5 s send limit |
| `chatty-chat/app/services/connection_registry.py` | `ConnectionRegistry` and `SocketGroups`: the sockets of each user, channel and team, first and last socket |
| `chatty-chat/app/clients/presence.py` | `PresenceStore`: who is online |
| `chatty-chat/app/services/event_bus.py` | `EventBus.publish`, `send_to_user`, `send_typing`, the Redis channel names |
| `chatty-chat/app/clients/core.py` | `CoreClient`: the cache, `MGET` and the Lua write |
| `chatty-chat/app/clients/core_events.py` | `mark_stale`, `mark_all_stale`, `clear_cache`, the listener loop |
| `chatty-chat/app/core/logging.py` | `RedactTokens`: hides `token=…` in uvicorn's log lines |
| `chatty-chat/tests/integration/test_two_gateways_real.py` | One event reaching sockets on two gateways |
| `chatty-chat/tests/integration/test_typing_real.py` | Typing reaching another gateway, but not the user's own tabs |
