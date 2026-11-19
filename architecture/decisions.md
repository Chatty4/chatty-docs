# Decisions

Why Chatty looks this way. The full list with status is in [chatty-infra/docs/decisions.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/decisions.md); this page explains the reasons in plain words.

## Summary

| # | Decision | Why, in one line |
|---|---|---|
| D-01 | Every ID is a UUID, in both services | Nobody can guess IDs by counting, and both services use the same type |
| D-02 | chatty-chat signs attachment download links itself, all the files of a message in one call | No call to chatty-core, one call per message, and the links work in `<img>` without a token |
| D-03 | chatty-chat calls chatty-core to attach a file before it saves the message, and to detach it if saving fails | The cleanup job never deletes a file that is in a sent message |
| D-04 | The web app un-hides a hidden DM when it gets a new message | No backend work and no extra call on every DM message |
| D-05 | Authors edit and delete their own messages; channel owners and team owners or admins can also delete | Same people who can manage the channel |
| D-06 | Every event from chatty-core has one envelope: `type`, `occurred_at`, `data` | One parser in chatty-chat |
| D-07 | One internal call returns all of a user's teams, channels, roles and settings | chatty-chat caches one answer and has everything it needs |
| D-08 | Events go over one Redis pub/sub channel, `core.events`; chatty-chat clears its cache on every (re)connect | Simplest option for two services; a missed event only lasts until the cache resets |
| D-09 | The client IP for rate limits comes from `X-Real-IP` set by Nginx, only when `BEHIND_PROXY=True` | Behind a proxy every request has the proxy's IP, and clients can fake headers |
| D-10 | Access tokens stay valid until they expire (15 minutes), even after logout | No lookup on every request in either service |
| D-11 | Any member can create invites that never expire | Owners and admins see and revoke every invite, and a member's invites are revoked when they leave |
| D-12 | `unread` goes to the user's other tabs on Redis pub/sub `chat.users:{user_id}` | A user's tabs can sit on different gateways, and a read position is private, so it can't go on the channel |
| D-13 | Push subscriptions live in chatty-chat | The push worker runs there: no call to chatty-core per message, and pushes work while chatty-core is down |
| D-14 | One MinIO bucket, separated by prefix (`uploads/`, `avatars/`, `team-icons/`, `attachments/`) | The service checks who may see a file before it signs a URL; prefixes still give per-prefix policies and lifecycle rules |
| D-15 | Upload to `uploads/{file_id}`, copy to the final key on confirm, check the copy | A presigned URL can't be revoked, so a late upload must never reach a file that was already checked |
| D-16 | URLs are signed for the public MinIO address; attachments download as `attachment` | The host is part of the signature, and an uploaded HTML or SVG file must never open as a page |
| D-17 | The hourly cleanup also deletes unsent attachments, unused avatars and icons, and files of deleted teams | Otherwise a confirmed file that is never used stays in MinIO forever |
| D-18 | No programs or installers, by name and type at upload and by content at confirm | A chat is an easy way to spread malware renamed to `report.pdf` |

**Status:** all accepted. **Built today:** all of them. The web app doesn't upload files yet, so D-02, D-03,
D-14 to D-18 are only reachable through the API so far.
