# Auth and security

How users and services prove who they are.

## User tokens (JWT RS256)

Every authenticated request to `/api/core/v1` carries an access token in the `Authorization: Bearer <token>` header.

| Property | Value |
|---|---|
| Algorithm | RS256 (asymmetric) |
| Lifetime | 15 minutes (`JWT_ACCESS_TOKEN_LIFETIME = 900`) |
| Claims | `sub` (user UUID), `iat`, `exp`, `jti` (random UUID per token) |
| Verification | chatty-core signs; chatty-core and chatty-chat both verify with the public key |

The private key never leaves chatty-core. chatty-chat receives only the public half, mounted read-only into its container.

**On the WebSocket** the token goes in the URL, `GET /api/chat/v1/ws?token=<access token>`, because
browsers can't set headers on a WebSocket. chatty-chat checks it once at connect (bad or expired: close
`4001`) and closes the socket with `4001` when its `exp` is reached. uvicorn logs URLs with the token in
its error log (every WebSocket) and in its access log (any request that has `?token=`), so chatty-chat's
`RedactTokens` log filter, on both loggers, replaces `token=…` with `token=***` before the line is written
(`chatty-chat/app/core/logging.py`).

## Refresh token rotation

A refresh token is a 64-character random hex string prefixed with `rt_`. It is never stored raw — only its SHA-256 hash is persisted.

**How rotation works:**

1. Client sends the current refresh token to `POST /api/core/v1/auth/refresh`.
2. chatty-core looks up the hash, verifies it is not expired and the user is active, then in one atomic transaction marks the old token as rotated (`revoked_at`) and creates a new one.
3. The response contains a new access token + new refresh token. The client replaces both.

**Reuse detection:** if a refresh token that was rotated more than 30 s ago (`JWT_REFRESH_REUSE_GRACE`) is replayed, chatty-core treats it as a theft signal and deletes _all_ refresh tokens of that user. Every device is logged out. Within the 30 s it is a retry after a lost answer: the token is rotated again and gets one more new pair. Only rotation sets `revoked_at`, so a rotated token stays in the table until it expires as the proof of a later reuse.

**Logout** (`POST /api/core/v1/auth/logout`) deletes the single refresh token supplied in the request body. The token must belong to the authenticated user; supplying another user's token returns 403.

**Change password** (`POST /api/core/v1/auth/password`) deletes all _other_ refresh tokens of the user, keeping the current device's token alive. All other devices are logged out.

## Service token (/internal)

chatty-chat calls chatty-core's `/internal` endpoints with a shared secret in the `X-Service-Token` header. The comparison uses `hmac.compare_digest` to prevent timing attacks.

The token is set in both services via the `CORE_SERVICE_TOKEN` environment variable, injected by docker-compose from `chatty-infra/.env`.

## Passwords and rate limits

**Password hashing:** PBKDF2-SHA256 (Django default). Password rules: minimum 10 characters, not entirely numeric, not too similar to the email or display name, not on the common-passwords list.

**Timing:** an unknown email and a wrong password both take ~1.2 s to respond, so the response time does not reveal whether an account exists. chatty-core runs a dummy `set_password` call for unknown emails to match the cost of a real `check_password`.

**Rate limits (all stored in Redis, keyed by the client's real IP behind Nginx):**

| Endpoint | Limit | Window | Key |
|---|---|---|---|
| `POST /auth/register` | 10 attempts | 1 hour | per client IP |
| `POST /auth/login` | 5 attempts | 15 minutes | per email and IP together, AND per IP (either triggers) |
| `POST /auth/password` | 5 attempts | 15 minutes | per user ID |
| `GET /invites/{invite_code}` and `POST /invites/{invite_code}/join` | 30 together | 1 minute | per client IP |
| `POST /files` | 30 attempts | 1 minute | per user ID |

Every attempt is counted before the password is checked, so parallel guesses all count. A success on
`/auth/login` clears the email-and-IP counter and takes back its own hit on the IP counter; a success on
`/auth/password` clears its counter.

## Secrets and keys

**JWT key pair (RSA-2048):** created once by `setup.bat` in `chatty-core/keys/`. The pair is never regenerated at container start — doing so would log out every user. The private key is only accessible inside the chatty-core container; chatty-chat gets a read-only volume mount of the public key.

```
chatty-core/keys/
  jwt_private.pem   → mounted at /app/keys/ in the core container
  jwt_public.pem    → mounted at /app/keys/ in core, and at /keys/ (read-only) in chat
```

**Other secrets** (`CORE_SECRET_KEY`, `CORE_SERVICE_TOKEN`, database passwords) are generated once by `setup.bat` and stored in `.env` files. They are never committed. `.env.example` lists every variable with a placeholder value.

## ⚠️ Known gaps

- **Access tokens cannot be revoked before expiry.** Logout deletes the refresh token, but any access token issued to that device remains valid for up to 15 minutes. This is standard for short-lived JWTs; a token blocklist is the mitigation when that window matters.
- **No 2FA / MFA.**
- **Tokens are not bound to IP or device.** A stolen access token can be used from any IP until it expires.
