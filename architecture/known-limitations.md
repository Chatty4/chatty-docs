# Known limitations

What is not right yet, grouped by theme.

## Auth

- **Access tokens cannot be revoked before expiry.** Logout and password-change revoke the refresh token, but the access token issued to that device stays valid for up to 15 minutes. A token blocklist (e.g. in Redis) is the standard mitigation when that window matters.
- **No 2FA / MFA.** Password-only authentication.
- **Tokens are not bound to IP or device.** A stolen access token can be replayed from any IP until it expires.
- **Rate limits are off while Redis is down.** chatty-core fails open (CHAT-254): login, register, password change and invite lookup and join keep working, but nothing limits password guesses until Redis is back. Each wrong login still costs ~1.2 s of hashing, and `/health` reports Redis as down.
- **One admin level.** Every staff user is a full admin in Django admin; there are no limited staff roles.

## Teams and channels

- **Deleting a team doesn't stop live events right away.** chatty-core sends only `team.deleted`, with no `membership.changed removed` for each channel, and chatty-chat doesn't unsubscribe open WebSockets from the team's channels for it yet. Members keep receiving those channels' events until they reconnect or their access token expires (up to 15 minutes). Removal from a channel, leaving or being removed from a team (CHAT-263) and deactivation are immediate ([CHAT-185](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-185)).
- **A user removed from a team can still watch its public channels until they reconnect.** chatty-chat checks a WebSocket `watch` against the memberships it loaded when the socket connected, so after leaving or being removed from a team, the user can start a new watch on that team's public channels and receive their live events until they reconnect or their access token expires (up to 15 minutes). A watch that is already running ends right away ([CHAT-231](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-231)).
- **No unarchive.** An archived channel stays read-only; there is no endpoint to undo it.

## Messages and live events

- **Channel events aren't cleaned up or sent to Kafka yet.** `channel_events` is meant to keep 30 days and to be the outbox for Kafka, but neither the cleanup nor the relay runs: the table only grows, and `published_at` stays null. Live events go over Redis pub/sub only.
- **A live event is lost while Redis is down.** The message is saved and its event row is written, but nobody hears it live. Clients get it from `POST /sync`; when Redis is back the gateway closes every socket with `1001` so they reconnect and sync.
- **One stuck socket delays its gateway's events.** The gateway sends to a channel's sockets together; a socket that stops reading holds up the next event for everyone on that process for up to 5 s, once, before it is dropped and closed with `1001 too_slow`. A queue per socket is the fix if the load test shows it ([CHAT-224](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-224)).
- **Sync makes up to two queries per channel**, for up to 500 channels in one call ([CHAT-224](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-224)).
- **No rate limits in chatty-chat.** Neither sending nor WebSocket messages are limited; `4008 too_many_messages` is in the contract but not sent.
- **A team admin can't delete messages in a private channel they aren't in.** They can't read the channel, so they get `404`, although D-05 lets team admins delete any message. To confirm with the PO.
- **Not built yet:** files on messages ([CHAT-212](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-212)), reactions and pins ([CHAT-127](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-127)), the read endpoint and unread counts ([CHAT-128](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-128)), typing and presence ([CHAT-215](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-215)).

## Infra

- **No HTTPS in the dev stack.** All traffic between the browser, Nginx, and the service containers is plain HTTP. TLS termination is expected to happen outside this stack in any non-dev environment, but where is not decided yet. Until then chatty-core sends no HSTS header, sets no `SECURE_SSL_REDIRECT` / `SECURE_PROXY_SSL_HEADER`, and CI doesn't run `manage.py check --deploy`.
