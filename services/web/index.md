# chatty-web

React app in the browser: login, teams, channels, DMs, invites, messages with files and GIFs, reactions,
pins, unread badges, presence dots, the typing indicator, search, push notifications, avatars and team
icons.

## At a glance

| | |
|---|---|
| Stack | Vite, React 19, TypeScript, React Router, Tailwind CSS 4, shadcn/ui (Base UI), Vitest |
| Calls | chatty-core `/api/core/v1`, chatty-chat `/api/chat/v1` and its WebSocket |
| Served by | The Vite dev server locally, Nginx in the Docker image. Both forward the API paths, so the browser only talks to its own origin and no CORS is needed |
| Stores | `localStorage`: the tokens, the session ID, the last used team, the theme. The browser keeps the push subscription and the service worker (`/sw.js`). Everything else lives in memory, per tab |
| Local address | <http://localhost:5173> (`npm run dev`) |
| Checks | `npm run lint`, `npm run typecheck`, `npm test`, `npm run build`, all four in CI (Node 24) |
| Settings | `VITE_VAPID_PUBLIC_KEY` for push notifications and `VITE_GIPHY_API_KEY` for the GIF picker; empty turns each off. `VITE_` variables are built into the app, so never secrets. Docker: `MINIO_PUBLIC_URL` (chatty-core's) for the CSP |

The whole journey, from opening the app to logging out: [from login to logout](../../architecture/flows/from-login-to-logout.md).

## Who talks to it

The browser opens the app and everything goes through one origin:

| Path in the browser | Forwarded to | Dev | Docker |
|---|---|---|---|
| `/api/core/...` | chatty-core | `CORE_API_TARGET` | `CORE_API_UPSTREAM` |
| `/api/chat/...`, also the WebSocket | chatty-chat | `CHAT_API_TARGET` | `CHAT_API_UPSTREAM` |
| `/health/core`, `/health/chat` | The backend's `/health`, for the `/status` page | same | same |
| Anything else | The app: Nginx answers `index.html`, and React Router picks the page | | |

Two things don't go through the proxy:

- **MinIO, through presigned URLs** (D-16): uploads go straight to it with the `PUT` URL from `POST /files`,
  and attachment images and downloads come from the URLs of `GET /messages/{id}/attachments`. Avatars and
  team icons come from `GET /files/{id}`, which redirects to MinIO; the app downloads them with the token
  and shows them as `blob:` URLs.
- **GIPHY** (`api.giphy.com` and its media hosts), for the GIF picker.

The Docker image's Content Security Policy allows exactly these besides its own origin and jsDelivr.

Nginx also sets `X-Real-IP` for the rate limits (D-09) and keeps the WebSocket open for up to an hour without
traffic. A backend that isn't running answers `502`; Nginx itself still starts.

## How it checks callers

There is no server of its own, so nothing here is a security check. The backends decide; the web app only
hides what they would refuse.

| Check | What it does | Code |
|---|---|---|
| `RequireAuth` | Pages inside the app. Anonymous users go to `/login`, which brings them back afterwards | `src/app/layouts/require-auth.tsx` |
| `GuestOnly` | `/login` and `/register`. A logged-in user goes to the app | `src/app/layouts/guest-only.tsx` |
| Message actions | Who sees Edit, Delete, Pin and React (D-05), from the same rules as chatty-chat | `src/features/messages/utils/permissions.ts` |

## What it does

| Area | What it does | Flow |
|---|---|---|
| Session | Both tokens in `localStorage`, shared by the tabs. A Web Lock makes sure only one tab rotates the refresh token, and `BroadcastChannel` tells the others about logins and logouts. The access token is refreshed a minute before it expires, and once more on a `401` | [refresh and logout](../../architecture/flows/refresh-and-logout.md) |
| API client | Adds the token, turns the contract's error body into an `ApiError` (with `retryAfter` from a `429`'s `Retry-After`), and repeats a request once after a refresh | |
| App shell | Team rail, sidebar and main area. `/` opens the last used team, a team opens its default channel, and someone with no team gets a welcome page to create or join one | [from login to logout](../../architecture/flows/from-login-to-logout.md) |
| Teams, channels, DMs, invites, profile | The screens for chatty-core's API: create, settings, members, browse, invite links. The sidebar reloads after every reconnect, and when an event or `unread` arrives for a channel it doesn't list (someone added me, or a new DM). Renames and archives follow `channel.updated` | [teams and invites](../../architecture/flows/teams-and-invites.md), [join, leave and owner hand-off](../../architecture/flows/channel-membership.md) |
| DMs in the sidebar | Ordered by their newest message: one `GET /channels/{id}/messages?limit=1` per DM after the sidebar loads, then every `message.created`, mine too. The ✕ on a DM hides it (`PUT /channels/{id}/me`, `hidden: true`) after marking it read. A hidden DM with unread messages comes back and is un-hidden in chatty-core, so my other devices show it too (D-04) | |
| Avatars, team icons | Account settings uploads my avatar (`PUT /me`), team settings the team icon (`PUT /teams/{id}`, owners and admins). Avatars show on messages, pins, member lists and the user menu; the icon on the team rail, the sidebar and the invite page. Each image is downloaded once per tab | [upload a file](../../architecture/flows/upload-a-file.md) |
| WebSocket | One socket per tab, opened when the app shell mounts. Heartbeat every 20 s and when the tab is shown or hidden, reconnect with backoff, `POST /sync` after every open, a new socket a minute before the token expires | [connect the WebSocket](../../architecture/flows/websocket.md) |
| Message store | One state per loaded channel, ordered by `seq`. Applies events by the rules in `events.md`: drops duplicates, applies the next one, fills gaps with `GET …/events`, reloads on `gap_too_large` | [open a channel, reconnect and sync](../../architecture/flows/reconnect-and-sync.md) |
| Sending | Optimistic: the message shows at once as pending, then as sent or failed with the reason. A retry reuses the `client_msg_id` and its `file_ids`, and a failed message can be dismissed. After a `429` it says "Not sent: too many messages." and counts the wait down live; Retry is disabled until it is over | [send a message](../../architecture/flows/send-message.md) |
| Files | Attach with the 📎 button, Ctrl+V or drag and drop, up to 10 per message. Each file uploads at once (3 steps, with progress and an image thumbnail); Send waits until all are ready. Files show inside the message bubble: images as one grid that opens a full-screen preview (← →, Download), other files as cards that download. Pins show the first two and "+N" | [upload a file](../../architecture/flows/upload-a-file.md), [send a message](../../architecture/flows/send-message.md) |
| GIFs | The GIF button searches GIPHY from the browser (trending first, more on scroll). A GIF is sent as a message whose whole body is its GIPHY link; the app shows such a message as the GIF, also in pins and search, and push says "Sent a GIF". chatty-chat sees plain text | |
| Messages | Markdown, mentions with autocomplete, emoji, edit, delete, drafts per channel, older messages on scroll. Opening a conversation puts the cursor in the message box. Jumping to a message loads a page around it (`around_seq`), and newer pages follow on scroll (`after_seq`) | [edit and delete](../../architecture/flows/edit-and-delete-message.md), [open a channel](../../architecture/flows/reconnect-and-sync.md) |
| Watch | A public channel the user opens without joining it is watched on the socket, again after every reconnect, so its new messages arrive live. The gateway ignores more than one `watch` a second, so the app keeps them at least 1.1 s apart: one that comes sooner waits for its turn, and only the newest waiting one is sent | [connect the WebSocket](../../architecture/flows/websocket.md) |
| Search | A panel in the channel header (`?search`): `GET /search` in the active team, with filters for a channel, a person and a date range. Results show the highlighted text, my own messages marked, and open the message in its channel | [search messages](../../architecture/flows/search-messages.md) |
| Push notifications | Settings, Notifications: turn on asks for permission, registers `/sw.js` and saves the subscription in chatty-chat; turn off and log out remove it. A session that ends any other way (expired, revoked, deactivated, logged out in another tab) unsubscribes the browser too, so the old user's previews stop. The service worker shows "New message" or "You were mentioned" and opens the channel on click | [get a push notification](../../architecture/flows/get-a-push-notification.md) |
| Reactions, pins | Chips with counts, an emoji picker, a pin mark and a pins panel. Reaction counts change from events only, so a reaction isn't counted twice | [react](../../architecture/flows/react-to-a-message.md), [pin](../../architecture/flows/pin-a-message.md) |
| Unread | Badges in the sidebar from `GET /unread`, on load, on a team switch and after every reconnect. New messages add to them, and `unread` messages keep my other tabs in step. A channel is marked read while its newest message is on screen | |
| Presence | Who is online in the active team, from `GET /teams/{team_id}/presence` and then the `presence` messages. Shows a green dot on message avatars and on DMs in the sidebar | [see who is online](../../architecture/flows/see-who-is-online.md) |
| Typing | The composer sends `typing` while there is text. Who is typing shows as a row after the last message, like a message with bouncing dots | [someone is typing](../../architecture/realtime.md#someone-is-typing) |
| Rate limits | chatty-chat's `429 rate_limited` says how long to wait, from `Retry-After`: edits, deletes, reactions, pins and search show "Try again in N seconds", a 21st different reaction shows the reaction limit, and a gap fill tries again after the wait. Login and the other chatty-core limits use the same wait text | [auth and security](../../architecture/auth-and-security.md#rate-limits) |

Providers wrap the app shell in this order, so each one can use the ones before it: `AuthProvider` (around
the whole app), then `WsProvider`, `MessagesProvider`, `TeamsProvider`, `ChannelsProvider`, `UnreadProvider`,
`PresenceProvider`, `TypingProvider`. All but the first only exist while the user is logged in, so logging out closes the socket and drops the messages in memory.

## If something is down

| Down | What happens |
|---|---|
| chatty-core, on load | "Chatty can't be reached" with Try again. The tokens are kept |
| chatty-core, while refreshing | The session is kept. Only `401` or `403` on the refresh ends it |
| chatty-chat | The socket reconnects with backoff, and sooner when the browser says it is online again. `POST /sync` catches up after it opens |
| The network | Sending fails, and the message shows why with a Retry button |
| Redis | No live events. They arrive after the next reconnect or event gap |
| `POST /sync` itself | It retries 3 times. Held events are then released, and the next gap fill catches up. A `429` isn't retried, because live messages wait for the sync |
| A rate limit (`429`) | The action shows how long to wait. A failed send counts down and can be retried once the wait is over; a gap fill retries on its own after the wait |

## ⚠️ Known gaps

- Being added to a channel shows in the sidebar only once a message arrives in it, after a reconnect or
  after a team change.
- The DM order costs one request per DM on every sidebar load.
- The GIPHY key is public (it's in the bundle), and searches go from the browser to GIPHY.
- The connection status isn't shown, so a dropped socket looks like a quiet channel.
- Drafts and failed messages are in memory, so a reload loses them.

Full list: [known limitations](../../architecture/known-limitations.md#web).

## Key code

| File | What is there |
|---|---|
| `src/main.tsx` | The entry point: theme, `AuthProvider`, router |
| `src/app/routes.tsx` | Every route |
| `src/app/layouts/app-layout.tsx` | The shell and the providers |
| `src/lib/session.ts` | The tokens: storage, Web Lock, refresh, other tabs |
| `src/lib/api.ts` | `coreApi`, `chatApi`, the retry after a refresh |
| `src/lib/ws/client.ts` | `WsClient`: one socket and its heartbeat |
| `src/lib/ws/sync-gate.ts` | Holds live messages back during a sync |
| `src/lib/ws/backoff.ts` | Reconnect delays, `retryWithBackoff` (never sooner than a `429`'s `Retry-After`), `retryAfterMs` |
| `src/lib/ws/watch-throttle.ts` | Keeps `watch` frames at least 1.1 s apart |
| `src/lib/retry-after.ts` | `waitText` ("12 seconds", "a minute") and `secondsUntil` for a countdown |
| `src/hooks/use-countdown.ts` | The seconds left until a time, updated while it runs |
| `src/features/auth/` | `useAuthState`: restore, login, register, logout |
| `src/features/ws/ws-provider.tsx` | Connect, reconnect, close codes, `POST /sync` |
| `src/features/messages/messages-provider.tsx` | The message store and everything that changes it |
| `src/features/messages/store/event-sequencer.ts` | Duplicates, next event, gap fills (retried after a `429`) |
| `src/features/messages/ui/send-failure.tsx` | Under a message that wasn't sent: why, the countdown, Retry and Dismiss |
| `src/features/messages/utils/errors.ts` | The texts for message, reaction, pin and send errors, `429` and `reaction_limit_reached` included |
| `src/features/messages/store/channel-reducer.ts`, `apply-event.ts` | Pure state changes, tested without a browser |
| `src/features/messages/utils/permissions.ts` | D-05 in the UI |
| `src/features/unread/` | The counts, the provider and the sidebar badge |
| `src/features/presence/` | Who is online: the store and the provider |
| `src/features/typing/store/typing.ts` | Who is typing per channel, the send limit and the label, tested without a browser |
| `src/features/typing/hooks/use-send-typing.ts` | Sends `typing`, and holds a change that comes too soon |
| `src/features/typing/ui/` | The provider, the row after the last message and the dots |
| `src/features/messages/hooks/use-watch-channel.ts` | `watch` and `unwatch` for a channel I haven't joined |
| `src/features/messages/utils/jump-to-message.ts` | Scroll to a message, loading it with `around_seq` first |
| `src/features/search/` | The search panel, the filters and the highlight |
| `src/features/files/` | The 3-step upload with progress, the checks before it, `useFileUploads`, `useFileImage` (avatars and icons), `ImageUploadRow` |
| `src/features/messages/ui/message-attachments.tsx`, `image-grid.tsx`, `image-lightbox.tsx` | A message's files, the image grid and the preview |
| `src/features/gifs/` | The GIPHY search, the picker and `isGifMessage` |
| `src/features/channels/ui/channels-provider.tsx`, `utils/dm-order.ts` | The sidebar list, hidden DMs and the DM order |
| `src/components/app/user-avatar.tsx` | An avatar image, or the initials |
| `src/features/push/` | Push: permission, subscribe, save, turn off, the settings section, `forgetPushSubscription` when a session ends without the logout button |
| `public/sw.js` | The service worker that shows push notifications |
| `nginx/default.conf.template` | The proxy, headers and the single-page fallback |
