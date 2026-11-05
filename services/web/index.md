# chatty-web

React app in the browser: login, teams, channels, DMs, invites, messages, reactions and pins. Later unread
counts, typing, presence dots and files.

## At a glance

| | |
|---|---|
| Stack | Vite, React 19, TypeScript, React Router, Tailwind CSS 4, shadcn/ui (Base UI), Vitest |
| Calls | chatty-core `/api/core/v1`, chatty-chat `/api/chat/v1` and its WebSocket |
| Served by | The Vite dev server locally, Nginx in the Docker image. Both forward the API paths, so the browser only talks to its own origin and no CORS is needed |
| Stores | `localStorage`: the tokens, the session ID, the last used team, the theme. Everything else lives in memory, per tab |
| Local address | <http://localhost:5173> (`npm run dev`) |
| Checks | `npm run lint`, `npm run typecheck`, `npm run build`, `npm test` |

The whole journey, from opening the app to logging out: [from login to logout](../../architecture/flows/from-login-to-logout.md).

## Who talks to it

The browser opens the app and everything goes through one origin:

| Path in the browser | Forwarded to | Dev | Docker |
|---|---|---|---|
| `/api/core/...` | chatty-core | `CORE_API_TARGET` | `CORE_API_UPSTREAM` |
| `/api/chat/...`, also the WebSocket | chatty-chat | `CHAT_API_TARGET` | `CHAT_API_UPSTREAM` |
| `/health/core`, `/health/chat` | The backend's `/health`, for the `/status` page | same | same |
| Anything else | The app: Nginx answers `index.html`, and React Router picks the page | | |

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
| API client | Adds the token, turns the contract's error body into an `ApiError`, and repeats a request once after a refresh | |
| App shell | Team rail, sidebar and main area. `/` opens the last used team, a team opens its default channel, and someone with no team gets a welcome page to create or join one | [from login to logout](../../architecture/flows/from-login-to-logout.md) |
| Teams, channels, DMs, invites, profile | The screens for chatty-core's API: create, settings, members, browse, invite links, hide a DM | [teams and invites](../../architecture/flows/teams-and-invites.md), [join, leave and owner hand-off](../../architecture/flows/channel-membership.md) |
| WebSocket | One socket per tab, opened when the app shell mounts. Heartbeat every 20 s and when the tab is shown or hidden, reconnect with backoff, `POST /sync` after every open, a new socket a minute before the token expires | [connect the WebSocket](../../architecture/flows/websocket.md) |
| Message store | One state per loaded channel, ordered by `seq`. Applies events by the rules in `events.md`: drops duplicates, applies the next one, fills gaps with `GET …/events`, reloads on `gap_too_large` | [open a channel, reconnect and sync](../../architecture/flows/reconnect-and-sync.md) |
| Sending | Optimistic: the message shows at once as pending, then as sent or failed with the reason. A retry reuses the `client_msg_id`, and a failed message can be dismissed | [send a message](../../architecture/flows/send-message.md) |
| Messages | Markdown, mentions with autocomplete, emoji, edit, delete, drafts per channel, older messages on scroll, jump to a message | [edit and delete](../../architecture/flows/edit-and-delete-message.md) |
| Reactions, pins | Chips with counts, an emoji picker, a pin mark and a pins panel. Reaction counts change from events only, so a reaction isn't counted twice | [react](../../architecture/flows/react-to-a-message.md), [pin](../../architecture/flows/pin-a-message.md) |

Providers wrap the app shell in this order, so each one can use the ones before it: `AuthProvider` (around
the whole app), then `WsProvider`, `MessagesProvider`, `TeamsProvider`, `ChannelsProvider`. The last four only
exist while the user is logged in, so logging out closes the socket and drops the messages in memory.

## If something is down

| Down | What happens |
|---|---|
| chatty-core, on load | "Chatty can't be reached" with Try again. The tokens are kept |
| chatty-core, while refreshing | The session is kept. Only `401` or `403` on the refresh ends it |
| chatty-chat | The socket reconnects with backoff, and sooner when the browser says it is online again. `POST /sync` catches up after it opens |
| The network | Sending fails, and the message shows why with a Retry button |
| Redis | No live events. They arrive after the next reconnect or event gap |
| `POST /sync` itself | It retries 3 times. Held events are then released, and the next gap fill catches up |

## ⚠️ Known gaps

- No unread badges, typing or presence dots yet: the WebSocket types exist, but nothing handles them, and
  `GET /teams/{team_id}/presence` isn't called ([CHAT-209](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-209),
  [CHAT-217](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-217),
  [CHAT-197](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-197)).
- The sidebar doesn't follow new channels and DMs: it reloads when the team changes.
- The web app never sends `watch`.
- The Attach file button does nothing yet ([CHAT-214](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-214)).
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
| `src/lib/ws/backoff.ts` | Reconnect delays |
| `src/features/auth/` | `useAuthState`: restore, login, register, logout |
| `src/features/ws/ws-provider.tsx` | Connect, reconnect, close codes, `POST /sync` |
| `src/features/messages/messages-provider.tsx` | The message store and everything that changes it |
| `src/features/messages/store/event-sequencer.ts` | Duplicates, next event, gap fills |
| `src/features/messages/store/channel-reducer.ts`, `apply-event.ts` | Pure state changes, tested without a browser |
| `src/features/messages/utils/permissions.ts` | D-05 in the UI |
| `nginx/default.conf.template` | The proxy, headers and the single-page fallback |
