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

Owned by chatty-chat. No tables yet. **Planned**: messages and what hangs off them, see
[api-chat.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/api-chat.md).

## Redis

Shared by both services. chatty-core uses it only for rate-limit counters today. Each key is a counter
that expires at the end of its window. If Redis is down, the limits are skipped (fail open).

| Key | Counts | Limit |
|---|---|---|
| `register:{ip}` | Registrations from one IP | 10 per hour |
| `login_fail:{email}` | Failed logins for one email | 5 per 15 minutes |
| `login_fail:{ip}` | Failed logins from one IP | 5 per 15 minutes |
| `change_pw_fail:{user_id}` | Wrong old passwords on password change | 5 per 15 minutes |
| `invite_lookup:{ip}` | Invite lookups and joins from one IP | 30 per minute |

`{ip}` is the client's real IP behind Nginx (D-09).

**Planned:** the `core.events` pub/sub channel (CHAT-172, D-08) and chatty-chat's `members:*` and
`channel:*` cache keys.

## MinIO

Runs in the dev stack, not used yet. **Planned:** avatars, team icons and message attachments (D-02, D-03).

## Kafka

Runs in the dev stack, not used yet. **Planned:** chatty-chat's channel events on the `chat.events` topic, see
[events.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/events.md).
