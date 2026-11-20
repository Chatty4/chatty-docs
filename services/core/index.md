# chatty-core

Django: users, auth, teams, channels, files.

## At a glance

| | |
|---|---|
| Stack | Python 3.14, Django 6.1, DRF with async views, uvicorn (ASGI) |
| Public API | `/api/core/v1`, contract in [api-core.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/api-core.md) |
| Internal API | `/internal` for chatty-chat, contract in [api-internal.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/api-internal.md) |
| Events | Publishes `core.events` on Redis, contract in [core-events.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/core-events.md) |
| Health | `GET /health` |
| Stores | Postgres `core_db`, Redis (rate limits, `core.events`), MinIO (avatars, team icons, attachments). See the [data map](../../architecture/data-map.md) |
| Local address | <http://127.0.0.1:8000> |

## Who talks to it

| Caller | Calls | Status |
|---|---|---|
| chatty-web (the browser) | `/api/core/v1` | works |
| Platform admins | `/admin/` (Django admin) | works |
| chatty-chat | `/internal`: a user's memberships, one channel with its members, attach and detach a message's files | works |
| The browser, directly | MinIO, with the presigned URLs chatty-core signs: `PUT` an upload, `GET` an avatar or team icon | works |
| chatty-chat | listens to `core.events` on Redis | works |

## How it checks callers

| Path | Check | Code |
|---|---|---|
| `/api/core/v1` | `Authorization: Bearer <access token>`, an RS256 JWT. A deactivated user gets `403 user_inactive` on every request | `authn/authentication.py` |
| `/internal` | `X-Service-Token` must equal `CORE_SERVICE_TOKEN` (compared with `hmac.compare_digest`) | `shared/authentication.py` |
| `/admin/` | Django session login, staff users only (`is_staff`) | Django admin |
| No check | `auth/register`, `auth/login`, `auth/refresh`, `GET invites/{invite_code}`, `/health` | |

Details: [auth and security](../../architecture/auth-and-security.md).

## What it does

| Area | What it does | Contract |
|---|---|---|
| Auth and profile | Register, login, refresh, logout, change password; `GET` / `PUT /me` with my teams and my avatar | `api-core.md` |
| Teams | Create (with a `general` channel), rename, set the icon, delete; directory and roles; remove members; leave (the last active owner can't) | `api-core.md` |
| Invites | Create, list and revoke invite links; open one and join the team and its default channels | `api-core.md` |
| Channels | Create, browse, rename, archive; join, leave, add and remove members | `api-core.md` |
| DMs | Open a DM with 1 to 7 team members; the same group of people always gets the same DM. Checked and created under the team lock, like channel create, join and add members (CHAT-284) | `api-core.md` |
| My channel settings | Notification level, muted, hidden (`PUT channels/{channel_id}/me`) | `api-core.md` |
| Files | Upload in three steps (D-15): `POST /files` gives a presigned `PUT` URL for `uploads/{file_id}`, the browser uploads straight to MinIO, `POST /files/{file_id}/confirm` copies it to its final key and checks it. `GET /files/{file_id}` redirects to an avatar or team icon; `DELETE` deletes my own file that is in no message | `api-core.md`, [upload a file](../../architecture/flows/upload-a-file.md) |
| Attach and detach (internal) | `POST /internal/files/attach` marks all the files of a message attached, or none (D-03); `POST /internal/files/detach` deletes the files of a message that was deleted or never saved | `api-internal.md` |
| Users (admin) | Deactivate users from Django admin. Users are never deleted. The admin's CSS and icons are collected into the image (`collectstatic`) and served by WhiteNoise, since uvicorn serves no static files | `users/admin.py`, `config/settings.py` |
| Cleanup | `python manage.py purge_refresh_tokens` deletes refresh tokens that expired more than 30 days ago (daily). `python manage.py clean_up_files` runs every hour from the `core` container (CHAT-168). It removes files nobody uses from MinIO and the database: uploads never confirmed, attachments not sent within 24 h, replaced avatars and team icons, and every file of a deleted team (D-17) | `authn/management/commands/`, `files/management/commands/` |

**Who can do what in a channel** (rules in `channels/access.py`):

| Action | Who |
|---|---|
| Read | Channel members; anyone in the team for a public channel |
| Write (post, add members) | Channel members, while the channel is not archived |
| Manage (rename, archive, remove members) | The channel owner, a team owner or a team admin. Never DMs |

**Every channel keeps an owner.** When the last owner of a private channel leaves or is removed, the
member who joined first becomes the owner. See the flow
[join, leave and owner hand-off](../../architecture/flows/channel-membership.md).

**Every team keeps an active owner.** Deactivated owners don't count as owners. When a team's only
active owner is deactivated, the active admin who joined first becomes owner, or else the active member
who joined first. Deactivating a user also revokes every open invite they created and all their
refresh tokens. See the flow [teams and invites](../../architecture/flows/teams-and-invites.md).

## If something is down

| Down | What happens | `/health` |
|---|---|---|
| Postgres | Every API call fails with `500` | `503`, `"db": "error"` |
| Redis | Everything keeps working, but rate limits are skipped (fail open) and `core.events` are not sent (logged; chatty-chat's cache expires within 5 minutes) | `503`, `"redis": "error"` |
| MinIO | Confirm, delete and the file cleanup fail; signing URLs makes no network call, so `POST /files` and downloads still answer. The cleanup keeps the rows and tries again the next hour | not checked |
| chatty-chat | Nothing in chatty-core breaks | not checked |

Each health check gives up after 2 seconds. Code: `config/views.py`.

## ⚠️ Known gaps

- `core.events` is pub/sub: an event sent while chatty-chat isn't subscribed is lost. chatty-chat clears
  its whole cache when it subscribes again (D-08).
- Rate limits are skipped while Redis is down (fail open).

Full list: [known limitations](../../architecture/known-limitations.md#teams-and-channels).

## Key code

| File | What is there |
|---|---|
| `authn/services.py` | `AuthnService`: issue, rotate and revoke tokens; refresh-token reuse detection |
| `authn/authentication.py` | `JWTAuthentication`: checks the Bearer token on every request |
| `users/services.py` | `UserService`: register, login, password change, profile, `deactivate` |
| `users/admin.py` | The `deactivate` admin action; delete is turned off |
| `channels/access.py` | Pure read / write / manage rules |
| `channels/services.py` | `ChannelService`: `require_access`, join, leave, members, rename, archive |
| `channels/repositories/` | Channel and member queries, the channel-row lock for leave and remove |
| `teams/services/` | `TeamService` (`teams.py`): teams, members, roles, owner hand-off. `InviteService` (`invites.py`): invites and join |
| `files/services.py` | `FileService`: create an upload, confirm, download, delete, attach, detach, the hourly cleanup |
| `files/checks.py` | What confirm reads from the bytes: the real image type, and programs (D-18) |
| `files/access.py` | Which file can be an avatar or a team icon |
| `users/internal/`, `channels/internal/`, `files/internal/` | The `/internal` endpoints for chatty-chat |
| `shared/storage.py` | MinIO: presigned `PUT` and `GET` URLs, head, read, copy, delete |
| `shared/events.py` | `publish`: sends one event on `core.events` after the change commits |
| `shared/redis.py` | `rate_limit`: Redis counters that fail open |
| `shared/pagination.py` | Cursor pagination used by every list |
