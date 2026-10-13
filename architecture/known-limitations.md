# Known limitations

What is not right yet, grouped by theme.

## Auth

- **Access tokens cannot be revoked before expiry.** Logout and password-change revoke the refresh token, but the access token issued to that device stays valid for up to 15 minutes. A token blocklist (e.g. in Redis) is the standard mitigation when that window matters.
- **No 2FA / MFA.** Password-only authentication.
- **Tokens are not bound to IP or device.** A stolen access token can be replayed from any IP until it expires.

## Profile

- **Teams are not implemented yet.** `GET /me` always returns `"teams": []`.

## Infra

- **No HTTPS in the dev stack.** All traffic between the browser, Nginx, and the service containers is plain HTTP. TLS termination is expected to happen outside this stack in any non-dev environment.
