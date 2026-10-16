# Known limitations

What is not right yet, grouped by theme.

## Auth

- **Access tokens cannot be revoked before expiry.** Logout and password-change revoke the refresh token, but the access token issued to that device stays valid for up to 15 minutes. A token blocklist (e.g. in Redis) is the standard mitigation when that window matters.
- **No 2FA / MFA.** Password-only authentication.
- **Tokens are not bound to IP or device.** A stolen access token can be replayed from any IP until it expires.

## Teams and channels

- **Changes are not pushed to chatty-chat yet.** chatty-core does not publish `core.events` until CHAT-172: joining, leaving, adding or removing channel members, renaming or archiving a channel, role changes and leaving or deleting a team. chatty-chat sees these changes only when its cache expires (TTL), so for that long someone removed from a channel can still receive its events.
- **No unarchive.** An archived channel stays read-only; there is no endpoint to undo it.
- **DMs and channel settings are not built yet.** Starting a DM and `PUT /channels/{channel_id}/me` (notifications, hide DM) return `404`.

## Infra

- **No HTTPS in the dev stack.** All traffic between the browser, Nginx, and the service containers is plain HTTP. TLS termination is expected to happen outside this stack in any non-dev environment.
