# Data map

What is stored where, and why.

## core_db (Postgres)

Owned by chatty-core. Nobody else reads or writes it. All IDs are UUIDs (D-01).

| Table | One row is | Worth knowing |
|---|---|---|
| `users` | An account | `email` is stored lowercased and is unique. Users are deactivated (`is_active = false`), never deleted. `is_staff` opens Django admin |
| `authn_refresh_tokens` | One logged-in device | Only the SHA-256 hash of the token is stored, never the token. `revoked_at` is set on logout, rotation and password change. `purge_refresh_tokens` deletes old ones |
| `teams` | A team | Deleting sets `deleted_at` (soft delete); a deleted team answers `404` everywhere |
| `team_members` | A user in a team | `role`: `owner`, `admin` or `member`. One row per user per team. `invite` records which link they joined with |
| `invites` | An invite link | `code` is 8 characters and unique. `max_uses = 0` is unlimited, `expires_at = null` never expires. Revoking sets `revoked_at` |
| `channels` | A channel or a DM | `kind`: `public`, `private` or `dm`. Names are unique per team (not for DMs). `is_default` channels are joined on invite. A DM has `dm_key`, the sorted member IDs, unique per team, so the same people always get the same DM. Archiving sets `archived_at` |
| `channel_members` | A user in a channel | `role`: `owner` or `member`. My settings live here: `notify_level` (`all`, `mentions`, `none`), `muted`, `hidden` |

## chat_db (Postgres)

Owned by chatty-chat. IDs from chatty-core (`channel_id`, `user_id`, `author_id`, `file_id`) are stored
without foreign keys (D-01).

| Table | One row is | Worth knowing |
|---|---|---|
| `channel_counters` | A channel's seq counter | `last_seq` is the last number handed out. Every send, edit and delete takes the next one with `INSERT … ON CONFLICT DO UPDATE SET last_seq = last_seq + 1`, which locks the row until the commit and creates it for a channel's first message. So `seq` is per channel, not global, without gaps or duplicates |
| `messages` | A message | `seq` is unique per channel; history pages by it. `(author_id, client_msg_id)` is unique, so a retried send returns the original. Deleting sets `deleted_at`, clears `body` and `mentions`, and keeps `id` and `seq` |
| `channel_events` | A change in a channel | One row per `message.created`, `message.edited`, `message.deleted`, with the same `seq` as the change. Read by `GET /channels/{id}/events` and `POST /sync`. Meant to be kept 30 days and to be the outbox for Kafka (`published_at` null until sent); neither the cleanup nor the relay runs yet |
| `attachments` | A file on a message | What chatty-core's `attach` returned (D-03). Removed with its message (on delete cascade). Nothing writes it yet (CHAT-212) |
| `read_state` | How far a user read a channel | `last_read_seq`, only ever moved forward. Today only sending moves it, for the sender; the read endpoint and unread counts are **planned** (CHAT-128) |

## Redis

Shared by both services. chatty-core uses it for rate-limit counters and to publish `core.events`,
chatty-chat for its cache and to send channel events between its processes.

chatty-core's keys are counters that expire at the end of their window. If Redis is down, the limits are
skipped (fail open).

| Key | Counts | Limit |
|---|---|---|
| `register:{ip}` | Registrations from one IP | 10 per hour |
| `login_fail:{email}` | Failed logins for one email | 5 per 15 minutes |
| `login_fail:{ip}` | Failed logins from one IP | 5 per 15 minutes |
| `change_pw_fail:{user_id}` | Wrong old passwords on password change | 5 per 15 minutes |
| `invite_lookup:{ip}` | Invite lookups and joins from one IP | 30 per minute |

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

| Pub/sub channel | Carries | Stored |
|---|---|---|
| `core.events` | chatty-core's 5 events (`membership.changed`, `team.member.changed`, `channel.updated`, `team.deleted`, `user.deactivated`), published after each change commits ([core-events.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/core-events.md), D-06, D-08) | No: a subscriber that isn't connected misses the event |
| `chat.events:{channel_id}` | chatty-chat's channel events (`message.created`, `message.edited`, `message.deleted`), published after the commit. Only gateways with a socket in the channel subscribe ([events.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/events.md)) | No, but every event is also in `channel_events`, so `POST /sync` recovers a missed one |

chatty-chat listens to `core.events` twice: the cache listener marks the keys an event makes stale (bump
the version, then delete the key), and the gateway updates open sockets. Every time the cache listener
subscribes (at start and after Redis comes back) it bumps `version:all` and deletes all `members:*` and
`channel:*` keys, because events sent while it wasn't subscribed are lost (D-08). Code:
`chatty-chat/app/clients/core_events.py`. More: [live events](realtime.md).

## MinIO

Runs in the dev stack, not used yet. **Planned:** avatars, team icons and message attachments (D-02, D-03).

## Kafka

Runs in the dev stack, not used yet. **Planned:** chatty-chat's channel events on the `chat.events` topic, see
[events.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/events.md). A relay will send the
`channel_events` rows whose `published_at` is null, in order. Until then the events go over Redis pub/sub.
