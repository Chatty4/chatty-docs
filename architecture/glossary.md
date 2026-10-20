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
| `core.events` | The Redis channel chatty-core will publish changes on, for chatty-chat (**planned**, CHAT-172) |
