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
| `channel_counters` | A channel's seq counter | `last_seq` is the last number handed out. Locked with `SELECT ... FOR UPDATE` to take the next one, so `seq` is per channel, not global |
| `messages` | A message | `seq` is unique per channel; history pages by it. `(author_id, client_msg_id)` is unique, so a retried send returns the original. Deleting sets `deleted_at` and keeps `id` and `seq` |
| `channel_events` | A change in a channel | Kept 30 days, read by `after_seq` and `POST /sync`. Also the outbox: `published_at` is null until the relay sends it |
| `attachments` | A file on a message | What chatty-core's `attach` returned (D-03). Removed with its message (on delete cascade) |
| `read_state` | How far a user read a channel | `last_read_seq`; unread = messages after it. Sending moves the sender's own position |

## Redis

Shared by both services. chatty-core uses it for rate-limit counters, chatty-chat for its cache.

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
chatty-core directly.

| Key | Holds | Expires |
|---|---|---|
| `members:{user_id}` | The user's memberships response: `is_active`, teams, channels (D-07) | 5 minutes |
| `channel:{channel_id}` | One channel with its members and their notification settings | 5 minutes |

**Planned:** the `core.events` pub/sub channel (CHAT-172, D-08), which clears these keys right away (CHAT-187).

## MinIO

Runs in the dev stack, not used yet. **Planned:** avatars, team icons and message attachments (D-02, D-03).

## Kafka

Runs in the dev stack, not used yet. **Planned:** chatty-chat's channel events on the `chat.events` topic, see
[events.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/events.md).
