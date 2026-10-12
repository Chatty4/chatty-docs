# Flows

A flow follows one user journey across services. Each page has a picture first, then numbered steps, then what happens when a step fails.

## The flows

| Flow | Question it answers | Services |
|---|---|---|
| [Register and login](register-and-login.md) | How does a new user get an account and their first tokens? | web, core, Redis |
| [Refresh and logout](refresh-and-logout.md) | How does a user stay logged in? How does one device log out? | web, core |
| [Change password](change-password.md) | Why does this device stay logged in while the others are logged out? | web, core |

Planned, once the features exist: teams and invites, channels and messages, file upload, events between core and chat.

## How to read a flow page

1. Look at the picture. It shows who calls whom, in order.
2. Read the numbered steps. Each step matches an arrow.
3. Read "If a step fails" to see what breaks and what does not.
4. Follow the links to the service pages and the code.
