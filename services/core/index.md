# chatty-core

Django: users, auth, teams, channels, files.

## At a glance

TODO

## Who talks to it

TODO

## How it checks callers

TODO

## What it does

| Area | What it does | Contract |
|---|---|---|
| Auth and profile | Register, login, refresh, logout, change password; `GET` / `PUT /me` with my teams | `api-core.md` |
| Teams | Create, rename, delete; directory and roles; leave (the last owner can't) | `api-core.md` |
| Invites | Create, list and revoke invite links; open one and join the team | `api-core.md` |
| Channels | Create, browse, rename, archive; join, leave, add and remove members | `api-core.md` |

**Who can do what in a channel** (rules in `channels/access.py`):

| Action | Who |
|---|---|
| Read | Channel members; anyone in the team for a public channel |
| Write (post, add members) | Channel members, while the channel is not archived |
| Manage (rename, archive, remove members) | The channel owner, a team owner or a team admin. Never DMs |

**Every channel keeps an owner.** When the last owner of a private channel leaves or is removed, the
member who joined first becomes the owner. See the flow
[join, leave and owner hand-off](../../architecture/flows/channel-membership.md).

## If something is down

TODO

## ⚠️ Known gaps

- Changes to teams and channels are not sent to chatty-chat yet (CHAT-172).
- No DMs and no per-channel settings yet.

Full list: [known limitations](../../architecture/known-limitations.md#teams-and-channels).

## Key code

| File | What is there |
|---|---|
| `channels/access.py` | Pure read / write / manage rules |
| `channels/services.py` | `ChannelService`: `require_access`, join, leave, members, rename, archive |
| `channels/repositories.py` | Channel and member queries, the channel-row lock for leave and remove |
| `teams/services.py` | `TeamService`: teams, members, roles, invites |
| `shared/pagination.py` | Cursor pagination used by every list |
