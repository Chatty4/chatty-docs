# Known limitations

What is not right yet, grouped by theme.

## Auth

- **A refresh token replayed within 30 s isn't caught.** The grace for a retry after a lost answer also gives whoever sends the rotated token a new pair, and both chains stay live.
- **Access tokens cannot be revoked before expiry.** Logout and password-change delete the refresh token, but the access token issued to that device stays valid for up to 15 minutes. A token blocklist (e.g. in Redis) is the standard mitigation when that window matters.
- **No 2FA / MFA.** Password-only authentication.
- **Tokens are not bound to IP or device.** A stolen access token can be replayed from any IP until it expires.
- **Rate limits are off while Redis is down.** Both services fail open (CHAT-254, CHAT-286): login, register, password change, invite lookup and join, sending, reactions, search and the rest keep working, but nothing limits password guesses or floods until Redis is back. Each wrong login still costs ~1.2 s of hashing, and `/health` reports Redis as down.
- **One admin level.** Every staff user is a full admin in Django admin; there are no limited staff roles.

## Teams and channels

- **Deleting a team doesn't stop live events right away.** chatty-core sends only `team.deleted`, with no `membership.changed removed` for each channel, and chatty-chat doesn't unsubscribe open WebSockets from the team's channels for it yet. Members keep receiving those channels' events until they reconnect or their access token expires (up to 15 minutes). Removal from a channel, leaving or being removed from a team (CHAT-263) and deactivation are immediate ([CHAT-185](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-185)).
- **No unarchive.** An archived channel stays read-only; there is no endpoint to undo it.
- **A deleted team's attachments stay listed in chatty-chat.** `clean_up_files` removes the files from MinIO, but chatty-chat's `attachments` rows stay. chatty-core answers `404` for the team, but chatty-chat still lets the channel's members read its history, and `GET /messages/{id}/attachments` signs links to objects that are gone: following one gives MinIO's `404`.

## Chat

- **One `/sync` can still cause 500 chatty-core calls.** Each unknown channel ID in a request is one chatty-core lookup (404s aren't cached). They run in parallel ([CHAT-268](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-268)), at most 20 at once ([CHAT-270](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-270)), and a user can send at most 20 syncs a minute ([CHAT-286](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-286)), but each one still does all the work.
- **The rate limits are per user, in fixed windows.** A user can send up to twice a limit around the end of a window, and many accounts together aren't limited. Connecting to the WebSocket isn't limited, and only `watch` and `typing` are limited on it; `4008 too_many_messages` is in the contract but not sent.
- **Only one user's reactions are capped.** A user can have 20 different emoji on a message, but a message has no overall limit.

## Messages and live events

- **The gateways don't read Kafka.** The outbox relay sends every channel event to Kafka, but the gateways still get them from Redis pub/sub, published by the API after the commit. Kafka only feeds the push worker for now.
- **The relay doesn't notice it lost its lock.** If its control connection drops, the advisory lock and `LISTEN` are gone but the loop goes on, so a second relay can start next to it. The poll still sends events; with no row locks (CHAT-285), both relays send the same rows, so Kafka gets duplicates, which the push dedupe absorbs.
- **A message deleted on its way to Kafka keeps its text there.** The relay doesn't lock the rows it sends (CHAT-285), so a delete in that moment doesn't change the record; Kafka keeps it for 7 days. The push worker skips deleted messages (CHAT-283), but a push that already went out can't be taken back.
- **The cleanups run from loops in the containers.** `clean_up_events` runs once a day in `chat`, `purge_refresh_tokens` once a day and `clean_up_files` every hour in `core`, and each timer restarts with its container. Production needs real scheduled jobs.
- **A live event is lost while Redis is down.** The message is saved and its event row is written, but nobody hears it live. Clients get it from `POST /sync`; when Redis is back the gateway closes every socket with `1001` so they reconnect and sync.
- **One stuck socket delays its gateway's events.** The gateway sends to a channel's sockets together; a socket that stops reading holds up the next event for everyone on that process for up to 5 s, once, before it is dropped and closed with `1001 too_slow`. A queue per socket is the fix if the load test shows it ([CHAT-224](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-224)).
- **Sync makes up to two queries per channel**, for up to 500 channels in one call ([CHAT-224](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-224)).
- **A socket can miss a channel removal that happens while it connects.** If the user is removed from a channel between loading their memberships and joining the channels, the socket still gets that channel's new events until it reconnects or its access token expires (up to 15 minutes) ([CHAT-271](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-271)).
- **Deleted text that was already sent stays with whoever got it.** A delete clears the text in the stored events, so sync and the events endpoint can't show it, but a tab that got `message.created` live keeps the text until it applies `message.deleted`.
- **A team admin can't delete messages in a private channel they aren't in.** They can't read the channel, so they get `404`, although D-05 lets team admins delete any message. To confirm with the PO.
- **Deleting an edited message blocks writes in its channel for a moment.** Clearing its text from `message.edited` events scans later events under the channel lock ([CHAT-272](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-272)). An index is the fix if [CHAT-224](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-224) shows it's needed.
- **A crash sends no `offline`.** If a process dies, its sockets never reach `disconnect`. The user's presence key expires after 60 s, so new tabs see them as offline, but open tabs keep showing them as online until the user's next change.
- **A page refresh can flicker offline and online,** also on the early reconnect before a token expires. A wait of about 5 s before `offline` would fix it.
- **A last close and a new connect at the same moment can end in the wrong order.** `offline` can land after `online`, and the user looks offline until they reconnect.
- **Unread mentions are capped at 100, like unread messages.** The contract states the cap only for `unread`; capping both keeps `mentions` at most `unread` ([CHAT-208](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-208)).
- **A detach that keeps failing leaves files attached to no message.** After a failed send it is tried once, after a delete three times, then logged; nothing retries it later.

## Attachment search

- **Attachment search only knows a fixed list of objects.** The labeler names what an image shows from a list of 35 words in `app/labeling/clip.py`; anything else gets the closest word from the list (a giraffe may be labeled "dog"). Changing the list needs a deploy, and files labeled before keep their old words until they are labeled again.
- **Attachment search misses some files.** Only images, PDF, DOCX, XLSX, PPTX and text files are read, and only their first 200,000 characters. Videos and archives match by file name only. A file that fails 3 times is never labeled.

## Web

- **Tokens are in `localStorage`.** The tabs need to share them, but any script running on the page can read them. The Docker image's Content Security Policy only allows the app's own scripts, which blocks injected ones, but the Vite dev server sets no policy.
- **A typing letter right after a message can show up to about 3 s late.** The server's 3 s limit still counts from the last `typing`, so the web app waits for it.
- **Each token renewal closes the socket before opening the next.** Together with the presence flicker above, a user with one tab can show as offline and online about every 14 minutes.
- **The sidebar doesn't hear about new channels and DMs directly.** No event says I was added, so the sidebar reloads when a message arrives for a channel it doesn't list, after a reconnect, or when the team changes. A channel I'm added to stays hidden until one of these.
- **Channels that aren't open only update their badge.** Their messages load from history when opened.
- **The DM order costs one request per DM.** chatty-core's sidebar list has no message times, so the web app asks chatty-chat for each DM's newest message (`limit=1`) on every sidebar load. A batch endpoint in chatty-chat is the fix if teams get many DMs.
- **The GIPHY key is public, and searches go to GIPHY.** The GIF picker calls GIPHY from the browser, so the key is in the bundle, anyone can use up its rate limit, and GIPHY sees what people search for and their IP.
- **A GIF is only a link to chatty-chat.** It is sent as a message whose body is its GIPHY link, so any other client, and chatty-chat's own search, sees the URL. If GIPHY removes the GIF, the message shows GIPHY's "not available" image.
- **A changed avatar or team icon shows after a reload.** Each image is downloaded once per tab, and nothing tells other people's tabs that it changed.
- **A removed upload waits for the cleanup.** A file taken out of the composer after it was uploaded isn't deleted; `clean_up_files` removes it after 24 hours.
- **The connection status isn't shown.** A dropped socket looks like a quiet channel.
- **Drafts and failed messages are in memory**, so a reload loses them.

## Infra

- **No HTTPS in the dev stack.** All traffic between the browser, Nginx, and the service containers is plain HTTP. TLS termination is expected to happen outside this stack in any non-dev environment, but where is not decided yet. Until then chatty-core sends no HSTS header, sets no `SECURE_SSL_REDIRECT` / `SECURE_PROXY_SSL_HEADER`, and CI doesn't run `manage.py check --deploy`.
- **The MinIO setup is only in `setup.bat`.** The bucket, the two service users and the rule that deletes `uploads/` after a day are created by `setup.bat` and `reset-db.bat`. Any other environment has to create them the same way; without the rule, objects written to an upload URL after confirm stay forever ([upload a file](flows/upload-a-file.md)).
