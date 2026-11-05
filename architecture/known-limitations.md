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

## Chat

- **`/sync` has no rate limit.** Each unknown channel ID in a request is one chatty-core lookup (404s aren't cached), so one request can cause up to 500 core calls. They run in parallel ([CHAT-268](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-268)), at most 20 at once ([CHAT-270](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-270)), so core never gets more than 20 at a time from one request, but it still does all the work.

## Messages and live events

- **Channel events aren't sent to Kafka yet.** `channel_events` is meant to be the outbox for Kafka, but the relay doesn't run: `published_at` stays null, and live events go over Redis pub/sub only. The nightly cleanup (CHAT-191) deletes by age only for now; once the relay runs, decide whether rows it never sent may be deleted.
- **The cleanup runs from a loop in the container.** It runs once a day after the `chat` container starts, and the 24 h timer restarts with the container. Production needs a real scheduled job.
- **A live event is lost while Redis is down.** The message is saved and its event row is written, but nobody hears it live. Clients get it from `POST /sync`; when Redis is back the gateway closes every socket with `1001` so they reconnect and sync.
- **One stuck socket delays its gateway's events.** The gateway sends to a channel's sockets together; a socket that stops reading holds up the next event for everyone on that process for up to 5 s, once, before it is dropped and closed with `1001 too_slow`. A queue per socket is the fix if the load test shows it ([CHAT-224](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-224)).
- **Sync makes up to two queries per channel**, for up to 500 channels in one call ([CHAT-224](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-224)).
- **No rate limits in chatty-chat.** Neither sending nor WebSocket messages are limited; `4008 too_many_messages` is in the contract but not sent.
- **A socket can miss a channel removal that happens while it connects.** If the user is removed from a channel between loading their memberships and joining the channels, the socket still gets that channel's new events until it reconnects or its access token expires (up to 15 minutes) ([CHAT-271](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-271)).
- **Deleted text that was already sent stays with whoever got it.** A delete clears the text in the stored events, so sync and the events endpoint can't show it, but a tab that got `message.created` live keeps the text until it applies `message.deleted`.
- **A team admin can't delete messages in a private channel they aren't in.** They can't read the channel, so they get `404`, although D-05 lets team admins delete any message. To confirm with the PO.
- **Deleting an edited message blocks writes in its channel for a moment.** Clearing its text from `message.edited` events scans later events under the channel lock ([CHAT-272](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-272)). An index is the fix if [CHAT-224](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-224) shows it's needed.
- **A crash sends no `offline`.** If a process dies, its sockets never reach `disconnect`. The user's presence key expires after 60 s, so new tabs see them as offline, but open tabs keep showing them as online until the user's next change.
- **A page refresh can flicker offline and online,** also on the early reconnect before a token expires. A wait of about 5 s before `offline` would fix it.
- **A last close and a new connect at the same moment can end in the wrong order.** `offline` can land after `online`, and the user looks offline until they reconnect.
- **Unread mentions are capped at 100, like unread messages.** The contract states the cap only for `unread`; capping both keeps `mentions` at most `unread` ([CHAT-208](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-208)).
- **Not built yet:** files on messages ([CHAT-212](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-212)), typing ([CHAT-216](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-216)).


## Web

- **Tokens are in `localStorage`.** The tabs need to share them, but any script running on the page can read them, and the Nginx config sets no `script-src` Content Security Policy.
- **No unread badges, typing or presence dots.** The WebSocket types exist, but nothing handles those frames, and `GET /teams/{team_id}/presence` isn't called ([CHAT-209](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-209), [CHAT-217](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-217), [CHAT-197](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-197)).
- **Each token renewal closes the socket before opening the next.** Together with the presence flicker above, a user with one tab can show as offline and online about every 14 minutes once dots are built.
- **The sidebar doesn't follow new channels and DMs.** Being added to one shows only after the team changes or the page reloads.
- **Events of channels that aren't open are ignored.** They load from history when opened, and nothing tells the user there is something new.
- **The web app never sends `watch`.**
- **The Attach file button does nothing yet** ([CHAT-214](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-214)).
- **The connection status isn't shown.** A dropped socket looks like a quiet channel.
- **Drafts and failed messages are in memory**, so a reload loses them.
- **CI doesn't run the tests.** `.github/workflows/ci.yml` runs lint, typecheck and build but not `npm test`, and uses Node 22 while the README and the Docker image use Node 24.

## Infra

- **No HTTPS in the dev stack.** All traffic between the browser, Nginx, and the service containers is plain HTTP. TLS termination is expected to happen outside this stack in any non-dev environment, but where is not decided yet. Until then chatty-core sends no HSTS header, sets no `SECURE_SSL_REDIRECT` / `SECURE_PROXY_SSL_HEADER`, and CI doesn't run `manage.py check --deploy`.
