# Data map

What is stored where, and why.

## core_db (Postgres)

Owned by chatty-core. Nobody else reads or writes it. All IDs are UUIDs (D-01).

| Table | One row is | Worth knowing |
|---|---|---|
| `users` | An account | `email` is stored lowercased and is unique. Users are deactivated (`is_active = false`), never deleted. `is_staff` opens Django admin |
| `authn_refresh_tokens` | One logged-in device | Only the SHA-256 hash of the token is stored, never the token. `revoked_at` is set only by rotation, so a rotated token stays as the proof of a later reuse. Logout, password change, reuse and deactivation delete rows. `purge_refresh_tokens` deletes tokens that expired more than 30 days ago |
| `teams` | A team | Deleting sets `deleted_at` (soft delete); a deleted team answers `404` everywhere |
| `team_members` | A user in a team | `role`: `owner`, `admin` or `member`. One row per user per team. `invite` records which link they joined with |
| `invites` | An invite link | `code` is 8 characters and unique. `max_uses = 0` is unlimited, `expires_at = null` never expires. Revoking sets `revoked_at` |
| `channels` | A channel or a DM | `kind`: `public`, `private` or `dm`. Names are unique per team (not for DMs). `is_default` channels are joined on invite. A DM has `dm_key`, the sorted member IDs, unique per team, so the same people always get the same DM. Archiving sets `archived_at` |
| `channel_members` | A user in a channel | `role`: `owner` or `member`. My settings live here: `notify_level` (`all`, `mentions`, `none`), `muted`, `hidden` |
| `files` | An avatar, a team icon or an attachment | `purpose`: `avatar` (no team), `team_icon` or `attachment` (a team), enforced by a check constraint. `status`: `pending` until confirm checks the upload, then `ready`. `object_key` is the final MinIO key. `attached_at` is set when a message holds it. Deleting sets `deleted_at`; the hourly cleanup removes the objects, then the row (D-17). `users.avatar` and `teams.icon` point here |

## chat_db (Postgres)

Owned by chatty-chat. IDs from chatty-core (`channel_id`, `user_id`, `author_id`, `file_id`) are stored
without foreign keys (D-01).

| Table | One row is | Worth knowing |
|---|---|---|
| `channel_counters` | A channel's seq counter | `last_seq` is the last number handed out. Every send, edit, delete, reaction change, pin and unpin takes the next one with `INSERT … ON CONFLICT DO UPDATE SET last_seq = last_seq + 1`, which locks the row until the commit and creates it for a channel's first message. So `seq` is per channel, not global, without gaps or duplicates |
| `messages` | A message | `seq` is unique per channel; history pages by it. `(author_id, client_msg_id)` is unique, so a retried send returns the original. Deleting sets `deleted_at`, clears `body` and `mentions`, and keeps `id` and `seq` |
| `channel_events` | A change in a channel | One row per `message.created`, `message.edited`, `message.deleted`, `reaction.added`, `reaction.removed`, `pin.added`, `pin.removed`, with the same `seq` as the change. Read by `GET /channels/{id}/events` and `POST /sync`. Rows are never changed, with one exception: deleting a message clears `body` and `mentions` in its earlier `message.created` and `message.edited` rows, so the deleted text can't be read back (CHAT-270); the `message.edited` rows are only searched if the message was edited (CHAT-272). Also the outbox for Kafka: `published_at` is null until the relay sent the row, and an insert sends `NOTIFY chat_events` to wake it. Kept 30 days: a nightly job (`python -m app.jobs.clean_up_events`, CHAT-191) deletes each channel's oldest events, up to its newest published `seq` older than 30 days, so a channel never loses events in the middle and no event is deleted before it reached Kafka |
| `reactions` | One user's emoji on one message | Primary key `(message_id, user_id, emoji)`: a user can react with several emoji, each once. `emoji` is `text` (one emoji can be several code points, up to 32 bytes). Deleting a message deletes its reactions in the same transaction, without `reaction.removed` events. Foreign key to `messages` (on delete cascade) |
| `pins` | A pinned message | Primary key `message_id`, so a message is pinned at most once. `pinned_by`, `pinned_at`; indexed by `(channel_id, pinned_at)` for the newest-first list. Max 100 per channel, counted under the channel's counter lock. Deleting a message deletes its pin in the same transaction, without `pin.removed`. Foreign key to `messages` (on delete cascade) |
| `attachments` | A file on a message | What chatty-core's `attach` returned (D-03): `file_id` (primary key), `object_key`, `filename`, `mime`, `size`, and `position`, the file's place in the sent `file_ids`. Written in the send transaction. `object_key` never leaves chatty-chat; it signs the download links (D-02). A deleted message keeps its rows but shows no files. Foreign key to `messages` (on delete cascade) |
| `read_state` | How far a user read a channel | `last_read_seq`, only ever moved forward (`GREATEST`). Sending moves it for the sender; `POST /channels/{id}/read` moves it, lowered to the channel's last `seq`. `GET /unread` counts the messages after it, capped at 100. No row means nothing read yet (CHAT-208) |
| `push_subscriptions` | One browser that wants Web Push (D-13) | `endpoint` is unique: one browser, one owner, so saving it again moves it to whoever is logged in. `p256dh` and `auth` encrypt the push for that browser. Each user keeps their 10 newest. The push worker deletes a row when the push service answers `404` or `410`. Never logged |

## Redis

Shared by both services. chatty-core uses it for rate-limit counters and to publish `core.events`,
chatty-chat for its cache and to send channel events between its processes.

chatty-core's keys are counters that expire at the end of their window. If Redis is down, the limits are
skipped (fail open).

| Key | Counts | Limit |
|---|---|---|
| `register:{ip}` | Registrations from one IP | 10 per hour |
| `login_fail:{email}:{ip}` | Logins for one email from one IP; cleared on success | 5 per 15 minutes |
| `login_fail:{ip}` | Logins from one IP; a success takes its own hit back | 5 per 15 minutes |
| `change_pw_fail:{user_id}` | Password changes of one user; cleared on success | 5 per 15 minutes |
| `invite_lookup:{ip}` | Invite lookups and joins from one IP | 30 per minute |
| `files_upload:{user_id}` | Started uploads (`POST /files`) of one user | 30 per minute |

`{ip}` is the client's real IP behind Nginx (D-09).

chatty-chat caches chatty-core's internal answers (api-internal.md, Caching). If Redis is down, it asks
chatty-core directly and caches nothing.

| Key | Holds | Expires |
|---|---|---|
| `members:{user_id}` | The user's memberships response: `is_active`, teams, channels (D-07) | 5 minutes |
| `channel:{channel_id}` | One channel with its members and their notification settings | 5 minutes |
| `version:members:{user_id}`, `version:channel:{channel_id}` | A counter an event increments when the key becomes stale | 1 day |
| `version:all` | A counter for "every key may be stale": `team.deleted` and the clear after a reconnect | never |

A cache miss reads the key's counter and `version:all` before it asks chatty-core, and stores the answer
only if neither changed meanwhile (a Lua script). So a request that read chatty-core just before a change
can't put the old answer back after the event deleted it (CHAT-266).

chatty-chat also keeps who is online here, because a process only knows its own sockets. These keys aren't
a cache: nothing else holds this data.

| Key | Holds | Expires |
|---|---|---|
| `presence:{user_id}` | Sorted set, one line per open socket: socket ID → time of its last heartbeat | 60 s after the last connect or heartbeat, so a crashed process leaves nothing behind |
| `online:{team_id}` | Set of the team's online user IDs, read by `GET /teams/{team_id}/presence` | Never; users leave it when their last socket closes |
| `visible:{user_id}` | Sorted set, one line per socket whose tab is on screen: socket ID → time of its last `visible: true` heartbeat. The push worker skips a user with a line from the last 45 s | 60 s after the last visible heartbeat |
| `push:{message_id}:{user_id}` | Set once when the push worker sends a message to a user, so a redelivered event isn't pushed twice | 1 day |

More: [see who is online](flows/see-who-is-online.md), [get a push notification](flows/get-a-push-notification.md).

| Pub/sub channel | Carries | Stored |
|---|---|---|
| `core.events` | chatty-core's 5 events (`membership.changed`, `team.member.changed`, `channel.updated`, `team.deleted`, `user.deactivated`), published after each change commits ([core-events.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/core-events.md), D-06, D-08) | No: a subscriber that isn't connected misses the event |
| `chat.presence:{team_id}` | `presence` frames (`online`, `offline`). Only gateways with a socket in the team subscribe | No: a tab that opens later asks `GET /teams/{team_id}/presence` |
| `chat.users:{user_id}` | Live messages for every socket of one user: `unread` after `POST /channels/{id}/read`. Only gateways with a socket of the user subscribe, by exact name (D-12) | No: the web app calls `GET /unread` after every reconnect |
| `chat.typing:{channel_id}` | `typing` frames, at most one per socket and channel every 3 s. Gateways subscribe to it together with `chat.events:{channel_id}` | No, and nothing recovers a missed one: the next frame comes 3 s later |
| `chat.events:{channel_id}` | chatty-chat's channel events (`message.created`, `message.edited`, `message.deleted`, `reaction.added`, `reaction.removed`, `pin.added`, `pin.removed`), published after the commit. Only gateways with a socket in the channel subscribe ([events.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/events.md)) | No, but every event is also in `channel_events`, so `POST /sync` recovers a missed one |

chatty-chat listens to `core.events` twice: the cache listener marks the keys an event makes stale (bump
the version, then delete the key), and the gateway updates open sockets. Every time the cache listener
subscribes (at start and after Redis comes back) it bumps `version:all` and deletes all `members:*` and
`channel:*` keys, because events sent while it wasn't subscribed are lost (D-08). Code:
`chatty-chat/app/clients/core_events.py`. More: [live events](realtime.md).

## MinIO

One bucket (`MINIO_BUCKET`, `chatty` locally), split by prefix (D-14). The browser never sends a file
through a service: it uploads and downloads with presigned URLs, signed for the address it reaches (D-16).

| Prefix | Holds | Written by |
|---|---|---|
| `uploads/{file_id}` | What the browser uploaded, until confirm copies it (D-15) | The browser, with the `PUT` URL from `POST /files`. A lifecycle rule (`minio/lifecycle.json` in chatty-infra) deletes these after a day |
| `avatars/{user_id}/{file_id}` | Avatars | chatty-core, on confirm |
| `team-icons/{team_id}/{file_id}` | Team icons | chatty-core, on confirm |
| `attachments/{team_id}/{file_id}` | Files on messages | chatty-core, on confirm |

Each service has its own MinIO user. chatty-core's can read, write and delete the whole bucket;
chatty-chat's can only read `attachments/*`, to sign download links (D-02). The policies are in
`chatty-infra/minio/`. More: [upload a file](flows/upload-a-file.md).

## Kafka

One topic, `chat.events`: chatty-chat's channel events, see
[events.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/events.md). 6 partitions, kept 7 days,
created by `setup.bat` (topics aren't created automatically).

| Producer | Consumer | Worth knowing |
|---|---|---|
| The outbox relay (`python -m app.jobs.relay`): the `channel_events` rows whose `published_at` is null, oldest first, then marks them | The push worker (group `push-worker`) | Key `channel_id`, so one channel's events stay in one partition, in order. Delivered at least once: a crash between sending and marking sends a batch again |

The gateways still get channel events from Redis pub/sub (`chat.events:{channel_id}`), published by the
API right after the commit; Kafka is for the workers. More:
[get a push notification](flows/get-a-push-notification.md).
