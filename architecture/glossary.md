# Glossary

Plain meanings for the hard words.

## Terms

| Term | Meaning |
|---|---|
| Team | A workspace: a group of people with their own channels. One person can be in many teams |
| Team role | `owner`, `admin` or `member`. Owners and admins manage the team; only owners delete it or make other owners |
| Channel | A room inside a team. **Public**: any team member can find and join it. **Private**: only people added to it |
| Default channel | A channel every new member joins automatically, like `general` |
| DM | A direct-message channel between 2 to 8 people of one team. Nobody can join, leave, add people to or rename it; it can only be hidden |
| Channel role | `owner` or `member`. A channel owner can rename, archive and remove people |
| Archived | Read-only. An archived channel can't get new messages or members |
| Invite code | The 8 characters at the end of an invite link. Whoever has the link can join the team |
| Deactivated | A user who can't log in or use the API any more. Their rows stay; users are never deleted |
| Access token | A short-lived (15 minutes) signed JWT sent on every request as `Authorization: Bearer` |
| Refresh token | A long-lived token used once to get a new token pair. One per logged-in device |
| Rotation | Every refresh replaces the refresh token; the old one stops working |
| Service token | The shared secret chatty-chat sends to chatty-core's `/internal` API in `X-Service-Token` |
| Rate limit | A cap on how often one IP, email or user can try something, counted in Redis |
| Fail open | When Redis is down, the rate limits are skipped instead of blocking everyone |
| Cursor pagination | Lists come in pages; the response has `next_cursor` to ask for the next page |
| Contract | The exact request and response of an endpoint or event, in chatty-infra/docs |
| `core.events` | The Redis channel chatty-core publishes membership, role, channel, team and deactivation changes on; chatty-chat listens, clears its cache and updates open WebSockets |
| `seq` | A channel's own counter. Every message, edit, delete and reaction change takes the next number, so `seq` gives the order of everything in a channel and shows gaps |
| `client_msg_id` | A UUID the web app makes for each message. Sending it again returns the first message instead of a second one |
| Channel event | One change in a channel (`message.created`, `message.edited`, `message.deleted`, `reaction.added`, `reaction.removed`) with its `seq`. Stored in `channel_events` and sent live |
| Reaction | An emoji a user puts on a message. One user can add several emoji to a message, each once; the message shows a `count` per emoji and `me` when I am one of them |
| Pub/sub | Redis channels that pass a message to whoever listens at that moment. Nothing is stored |
| Gateway | The part of each chatty-chat process that holds its WebSockets: one Redis subscription, forwarding events to the local sockets |
| Heartbeat | The message the web app sends every 20 s on the WebSocket. None for 60 s closes the socket |
| Watch | Following one public channel on the WebSocket without joining it |
| Sync | Getting the events missed while offline: `POST /sync` with the last `seq` of each channel |
| Cache hit / miss | Hit: the answer is in Redis. Miss: it isn't, so chatty-chat asks chatty-core and stores it |
| Version counter | A Redis counter per cached key that events increment; a cache write is refused if it changed meanwhile |
| Close code | The number a WebSocket is closed with; it tells the web app whether to refresh the token, log out or just reconnect |
