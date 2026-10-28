# Flows

A flow follows one user journey across services. Each page has a picture first, then numbered steps, then what happens when a step fails.

## The flows

| Flow | Question it answers | Services |
|---|---|---|
| [Register and login](register-and-login.md) | How does a new user get an account and their first tokens? | web, core, Redis |
| [Refresh and logout](refresh-and-logout.md) | How does a user stay logged in? How does one device log out? | web, core |
| [Change password](change-password.md) | Why does this device stay logged in while the others are logged out? | web, core |
| [Join, leave and owner hand-off](channel-membership.md) | Who can join or leave a channel, and who owns it after the owner leaves? | web, core |
| [Teams and invites](teams-and-invites.md) | How does a team start, and how does a new person get in with a link? | web, core, Redis |
| [Connect the WebSocket](websocket.md) | How does a tab stay connected, and what does each close code mean? | web, chat, core |
| [Send a message](send-message.md) | How does a message get its `seq`, and why does a retry not send it twice? | web, chat, core, Redis |
| [Edit and delete a message](edit-and-delete-message.md) | Who can change a message, and what stays when it is deleted? | web, chat, core, Redis |
| [Open a channel, reconnect and sync](reconnect-and-sync.md) | How does a tab catch up after a gap or a lost connection? | web, chat |
| [A change in chatty-core reaches chatty-chat](core-events.md) | How do removed users lose access at once, in the cache and on open sockets? | core, Redis, chat |

How the live parts fit together (sockets, Redis pub/sub, gateways, the cache): [live events](../realtime.md).

Planned, once the features exist: file upload, reactions and pins.

## How to read a flow page

1. Look at the picture. It shows who calls whom, in order.
2. Read the numbered steps. Each step matches an arrow.
3. Read "If a step fails" to see what breaks and what does not.
4. Follow the links to the service pages and the code.
