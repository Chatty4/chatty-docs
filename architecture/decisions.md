# Decisions

Why Chatty looks this way. The full list with status is in [chatty-infra/docs/decisions.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/decisions.md); this page explains the reasons in plain words.

## Summary

| # | Decision | Why, in one line |
|---|---|---|
| D-01 | Every ID is a UUID, in both services | Nobody can guess IDs by counting, and both services use the same type |
| D-02 | chatty-chat signs attachment download links itself | One request per download, no call to chatty-core |
| D-03 | chatty-chat calls chatty-core to attach a file before it saves the message, and to detach it if saving fails | The cleanup job never deletes a file that is in a sent message |
| D-04 | The web app un-hides a hidden DM when it gets a new message | No backend work and no extra call on every DM message |
| D-05 | Authors edit and delete their own messages; channel owners and team owners or admins can also delete | Same people who can manage the channel |
| D-06 | Every event from chatty-core has one envelope: `type`, `occurred_at`, `data` | One parser in chatty-chat |
| D-07 | One internal call returns all of a user's teams, channels, roles and settings | chatty-chat caches one answer and has everything it needs |
| D-08 | Events go over one Redis pub/sub channel, `core.events`; chatty-chat clears its cache on every (re)connect | Simplest option for two services; a missed event only lasts until the cache resets |
| D-09 | The client IP for rate limits comes from `X-Real-IP` set by Nginx, only when `BEHIND_PROXY=True` | Behind a proxy every request has the proxy's IP, and clients can fake headers |
| D-10 | Access tokens stay valid until they expire (15 minutes), even after logout | No lookup on every request in either service |
| D-11 | Any member can create invites that never expire | Owners and admins see and revoke every invite, and a member's invites are revoked when they leave |

**Status:** all accepted. **Built today:** D-01, D-09, D-10, D-11. The others wait for files, chatty-chat or
CHAT-172.
