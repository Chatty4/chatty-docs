# Flow: send a message

> **In one minute:** The web app sends a message with REST, never over the WebSocket. chatty-chat checks
> that the user can write in the channel, then in one transaction takes the channel's next `seq`, saves the
> message and an event row, and moves the sender's read position. Only after the commit it publishes
> `message.created`, and every open tab in the channel gets it. The web app makes a `client_msg_id` for
> each message, so a retry after a lost answer returns the same message instead of sending it twice.

## The picture

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant W as chatty-web
    participant H as chatty-chat
    participant C as chatty-core
    participant DB as chat_db
    participant R as Redis

    U->>W: Send "deploy at 5?"
    W->>H: POST /api/chat/v1/channels/{channel_id}/messages (Bearer, client_msg_id)
    H->>DB: Already sent with this client_msg_id?
    alt found, same channel
        H-->>W: 200 the original message
    else not found
        H->>R: cached channel (members, archived)
        opt cache miss
            H->>C: GET /internal/channels/{channel_id}
        end
        alt not a member / archived / no channel
            H-->>W: 403 / 409 / 404
        else can write
            H->>DB: BEGIN, next seq (locks the counter row)
            H->>DB: INSERT message, INSERT channel_events row, move my read position
            H->>DB: COMMIT
            H->>R: PUBLISH chat.events:{channel_id} message.created
            H-->>W: 201 the message object
        end
    end
```

## Step by step

1. **Validate.** `client_msg_id` is a UUID, `body` is at most 4000 characters, at most 10 `file_ids`. An
   empty body without files is `400 validation_error`. A body of only spaces counts as text.
2. **Retry check, first.** chatty-chat looks up `(author_id, client_msg_id)`. If the message exists in this
   channel, it returns it with `200`: no new `seq`, no event. This runs before the access check, so a retry
   still gets its message after the channel was archived. The same `client_msg_id` in another channel is a
   client bug: `400 validation_error` with `fields.client_msg_id: "already_used"`.
3. **Access.** The channel comes from the cache or chatty-core's `/internal`. The user must be a member,
   and the channel must not be archived or in a deleted team.
4. **Files.** Not available yet: a non-empty `file_ids` is `400 invalid_file`.
5. **Mentions.** Every `<@user_id>` in the body that is a member of the channel goes into `mentions`, once,
   in order. Mentions of other users stay in the text only.
6. **One transaction:**
   - The next `seq`: `INSERT … ON CONFLICT DO UPDATE SET last_seq = last_seq + 1` on `channel_counters`.
     It locks the counter row until the commit, so the seqs of one channel are handed out one at a time,
     and it also creates the row for a channel's first message.
   - The message row, with that `seq`.
   - A `channel_events` row: `message.created` with the same `seq` and the message object as payload.
   - The sender's read position moves to this `seq` (it only ever moves forward).
7. **After the commit**, `message.created` is published on `chat.events:{channel_id}`. Every gateway with
   a socket in the channel forwards it (see [live events](../realtime.md#from-a-saved-change-to-every-tab)).
8. `201` with the message object.

**Two sends with the same `client_msg_id` at the same moment** (a double-click). Both can pass step 2
before either saves. The first commits. The second waits for the counter lock, then hits the unique
`(author_id, client_msg_id)`, rolls back (so its `seq` isn't used and there is no gap), looks the message
up again and returns it with `200`. Neither gets a `500`.

**Why lock the counter row?** Without it, two sends could read the same `last_seq` and both take `seq` 9.
The upsert makes the second wait until the first commits, so 50 sends at once get exactly 1 to 50.

## If a step fails

| Step | What happens |
|---|---|
| No or bad token | `401 unauthorized` |
| The user is deactivated | `403 user_inactive` |
| Body empty without files, over 4000 characters, more than 10 files | `400 validation_error` |
| `client_msg_id` already used in another channel | `400 validation_error`, `client_msg_id: already_used` |
| `file_ids` given | `400 invalid_file` (files come with [CHAT-212](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-212)) |
| Not a member of the channel | `403 not_a_member` |
| The channel doesn't exist | `404 channel_not_found` |
| Archived, or its team was deleted | `409 channel_archived` |
| chatty-core unreachable and nothing cached | `503 core_unavailable` |
| DB error in the transaction | `500`; message, event row, seq and read position all roll back |
| Redis down when publishing | Logged; still `201`. Other tabs get the message from `POST /sync` ([reconnect and sync](reconnect-and-sync.md)) |

## ⚠️ Known gaps

- **No files yet.** Attaching through chatty-core (D-03) comes with
  [CHAT-212](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-212).
- **No rate limit on sending.**

## Key code

| File | What is there |
|---|---|
| `chatty-chat/app/api/routers/messages.py` | `send_message`: `201`, or `200` for a retry |
| `chatty-chat/app/services/message_service.py` | `MessageService.send`, `_retry`, `extract_mentions`, `_publish` |
| `chatty-chat/app/repositories/channel_counters.py` | `next_seq`: the locking upsert |
| `chatty-chat/app/repositories/messages.py` | `get_by_client_msg_id`, `insert_message` (a duplicate becomes `MessageAlreadySent`) |
| `chatty-chat/app/repositories/read_state.py` | `advance_read_position`: forward only (`GREATEST`) |
| `chatty-chat/tests/integration/test_messages_concurrency.py` | 50 sends at once, and two identical sends at once |
