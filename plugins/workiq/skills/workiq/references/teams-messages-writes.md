# Teams messages and writes

Use **Finding Teams targets** in `references/teams-routing.md` to resolve the chat, channel, or
message. Use this file for sending, replies, mentions, reactions,
reply-with-quote, edits, hiding chats, read state, and removal.

## Write paths

| Operation | Tool | Path |
| --- | --- | --- |
| Find, create, or reuse a 1:1 chat | `create_entity` | parentUrl `/chats` |
| Send a chat message | `create_entity` | parentUrl `/chats/{chatId}/messages` |
| Send a message to yourself | `create_entity` | parentUrl `/chats/48:notes/messages` |
| Reply to a chat message with a quote | `do_action` | `/chats/{chatId}/messages/replyWithQuote` |
| Post a channel message | `create_entity` | parentUrl `/teams/{teamId}/channels/{channelId}/messages` |
| Reply to a channel message | `create_entity` | parentUrl `/teams/{teamId}/channels/{channelId}/messages/{messageId}/replies` |
| Edit a chat message | `update_entity` | `/chats/{chatId}/messages/{messageId}` |
| Edit a channel message | `update_entity` | `/teams/{teamId}/channels/{channelId}/messages/{messageId}` |
| Remove a user-authored chat message | `do_action` | POST `/users/{userId}/chats/{chatId}/messages/{chatMessageId}/softDelete` |
| Remove a channel message | `do_action` | POST `/teams/{teamId}/channels/{channelId}/messages/{messageId}/softDelete` |
| React to a message | `do_action` | `/chats/{chatId}/messages/{messageId}/setReaction` (or the channel-message equivalent) |
| Remove my reaction | `do_action` | `/chats/{chatId}/messages/{messageId}/unsetReaction` (or the channel-message equivalent) with `{"reactionType":"{reactionToRemove}"}` (literal Unicode emoji, e.g. `👍`) |
| Pin a chat message | `create_entity` | parentUrl `/chats/{chatId}/pinnedMessages` with `{"message@odata.bind":"https://graph.microsoft.com/v1.0/chats/{chatId}/messages/{messageId}"}` |
| Unpin a chat message | `delete_entity` | `/chats/{chatId}/pinnedMessages/{pinnedMessageId}` |
| Remove (hide) a chat from my list | `do_action` | `/chats/{chatId}/hideForUser` |
| Mark a chat read or unread | `do_action` | `/chats/{chatId}/markChatReadForUser`, `/chats/{chatId}/markChatUnreadForUser` |

Message body shape (chat and channel):
`{"body": {"contentType": "text", "content": "..."}}`.

Do not add `@odata.type` to the message or its `body` when creating or
editing messages; Graph rejects `#microsoft.graph.itemBody` on a chat message
body with HTTP 400. When sending a chat or channel message, do not call
`get_schema`, because the generated schema adds these annotations (the
create-properties question at the end of this file is the only exception).

## Posting an important channel announcement

For "broadcast with importance", "urgent", or "important" channel posts,
resolve the channel with **Finding Teams targets** and call `create_entity` once on
`/teams/{teamId}/channels/{channelId}/messages` with:

```json
{"subject":"{subject}","importance":"high","body":{"contentType":"html","content":"<at id=\"0\">{mentionText}</at> {message}"},"mentions":[{"id":0,"mentionText":"{mentionText}","mentioned":{"user":{"id":"{directoryUserId}","displayName":"{mentionText}","userIdentityType":"aadUser"}}}]}
```

Omit `mentions` and the `<at>` tag when nobody is tagged. Use the returned
message `id` for any follow-up reaction or reply.

## Sending a message to yourself

Call `create_entity` directly with parentUrl `/chats/48:notes/messages`. Do not
list chats, look up users, or create a chat first.

## Sending a message to a person — reuse the existing chat

To "send a chat to Alex":

1. Find the chat **by person**. Graph returns the existing chat when one
   already exists and creates it only when needed.
2. POST the message to that chat with `create_entity` on `/chats/{chatId}/messages`.
3. Never create a group chat to deliver a single 1:1 message.

When adding a member to a chat, use a supported role such as `owner`; never
send `roles: []` for chat members. (Team and channel member adds do accept
`roles: []` for a regular member.)

## Reading a channel thread

For a channel thread identified by its root message text:

1. **Finding Teams targets** to resolve the exact team and channel once.
2. Fetch the channel message collection once and select the root locally by
   exact body text.
3. Fetch `/teams/{teamId}/channels/{channelId}/messages/{rootId}/replies`.
4. Fetch channel members only when a participant-vs-silent comparison is
   required, then stop.

Do not re-fetch the root when the collection already contains the needed
fields, try another same-name channel after confirmation, fall back to `ask`,
use `/chats/{channelId}/...` for channel reads, or repeat collection or reply
reads.

## Replying with a Teams mention

Resolve the mentioned user's directory ID (see
`references/teams-members-presence.md`), then create the reply on the exact
channel thread with this payload:

```json
{"body":{"contentType":"html","content":"Summary <at id=\"0\">{mentionText}</at>"},"mentions":[{"id":0,"mentionText":"{mentionText}","mentioned":{"user":{"id":"{directoryUserId}","displayName":"{mentionText}","userIdentityType":"aadUser"}}}]}
```

The `<at id>` marker, `mentions[0].id`, `mentionText`, and `mentioned.user.id`
must all identify the same person. Do not call `get_schema` for this recipe.

## Reacting to a message

1. Find the target: **Finding Teams targets** for the channel or chat.
2. Get the exact message `id` from the message lookup there.
3. Call `do_action` on the matching path with `{"reactionType":"👍"}` (or the
   requested emoji):
   - Channel: `/teams/{teamId}/channels/{channelId}/messages/{messageId}/setReaction`
   - Chat: `/chats/{chatId}/messages/{messageId}/setReaction`

Send the literal Unicode reaction, for example `{"reactionType":"👍"}`; never
`"like"`. The action schema types this as a plain string, so do not call
`get_schema` to infer accepted values.

## Replying to a chat message with a quote

Use `replyWithQuote` when the user asks to quote, cite, or reply to a specific
chat message so the source appears in the reply. It applies to chats only; for
channel threads, POST to `/replies` instead.

1. Resolve the exact chat with **Finding Teams targets**.
2. Fetch `/chats/{chatId}/messages?$select=id,createdDateTime,from,body` once
   and select the source locally. Match exact text as in **Finding Teams targets**;
   when the user identifies the source by sender and recency ("their last
   message about X"), take that sender's latest message by `createdDateTime`
   and confirm its body matches the described topic. If it does not, report
   that rather than quoting an older or different message.
3. Call `do_action` once on `/chats/{chatId}/messages/replyWithQuote` with
   exactly this body shape:

   ```json
   {"messageIds":["{messageId}"],"replyMessage":{"body":{"contentType":"text","content":"{reply}"}}}
   ```

Include only the selected message IDs in `messageIds` (at most 10). Do not add
`@odata.type` to `replyMessage` or its `body`, wrap the body in
`replyWithQuoteMessagePayload`, or change property casing; each of these
returns HTTP 400. Do not post a plain message as a fallback, call
`search_paths` or `get_schema`, or use `ask` to find the source. Confirm from
the returned message ID.

## Edit a message

### Edit a chat message

Use **Finding Teams targets** to resolve the target and message, and call `update_entity`
on `/chats/{chatId}/messages/{messageId}`.

### Edit a channel message

Use **Finding Teams targets** to resolve the target and message, and call `update_entity`
on `/teams/{teamId}/channels/{channelId}/messages/{messageId}`.

For both surfaces, send only
`{"body":{"contentType":"text","content":"..."}}` as the update body. Never
include `@odata.type`, even if a generated entity schema marks it required. Do
not call `search_paths` or `get_schema` for these known edit paths.

## Removing a Teams message

Do not call `delete_entity` for chat or channel messages. Discovery metadata
may advertise DELETE paths that the deployed runtime rejects as unsupported.

For a user-authored chat message, call `do_action` once with POST
`/users/{userId}/chats/{chatId}/messages/{chatMessageId}/softDelete`, using the
message sender's user ID and an empty JSON body.

For a channel message, resolve the team, channel, and exact message with
**Finding Teams targets**, then call `do_action` once with POST
`/teams/{teamId}/channels/{channelId}/messages/{messageId}/softDelete` and an
empty JSON body. For a channel reply, append `/replies/{replyId}` before
`/softDelete`.

If the action returns a policy or access-denied response, stop: do not retry
raw versus encoded IDs, fall back to `delete_entity`, or probe alternate delete
paths.

## Removing/Deleting/Hiding a chat from the current user's chat list

"Delete this chat from my list", "remove this chat", and "hide this chat" map
to the per-user `hideForUser` action, never `delete_entity`.

1. Find the exact chat with **Finding Teams targets**. For a named topic, use the
   single batched topic lookup documented there; for a named person, use the
   1:1 resolver.
2. Resolve your member identity. For a topic lookup, reuse the expanded member
   whose `userId` matches the signed-in user.
3. Call `do_action` on `/chats/{chatId}/hideForUser` with:

   ```json
   {"user":{"@odata.type":"#microsoft.graph.teamworkUserIdentity","id":"{signedInUserId}","tenantId":"{signedInMemberTenantId}","userIdentityType":"aadUser"}}
   ```

4. Stop after the successful `204` response.

For a topic lookup, do not issue a separate `/me` or
`/chats/{chatId}/members` fetch. Do not make a verification fetch.

## Marking a named 1:1 chat read or unread

This is a known deployed contract. Do not call `search_paths` or `get_schema`.

For mark-read, the sequence is exactly:

1. `fetch` the signed-in user and exact counterpart with the 1:1 chat lookup in
   **Finding Teams targets**.
2. `create_entity` on `/chats` to create or return the one-on-one chat.
3. `fetch` exactly `/chats/{chatId}/members` with no query string and resolve
   your member identity from its `userId` and `tenantId`.
4. `do_action` on `/chats/{chatId}/markChatReadForUser`.

For mark-unread, use the same first three steps, then fetch
`/chats/{chatId}/messages?$select=createdDateTime` and call
`/chats/{chatId}/markChatUnreadForUser` with the first returned message
timestamp. The chat resource's `lastUpdatedDateTime` is not a message timestamp
and is not a valid substitute.

Both actions use the same `user` object as `hideForUser`:

| Intent | Body |
|--------|------|
| Mark read | `{"user":{"@odata.type":"#microsoft.graph.teamworkUserIdentity","id":"{signedInUserId}","tenantId":"{signedInMemberTenantId}","userIdentityType":"aadUser"}}` |
| Mark unread | The same `user` object plus `"lastMessageReadDateTime":"{returnedCreatedDateTime}"` beside it |

Do not call `search_paths` or `get_schema`, omit `tenantId`, or probe
unsupported member fields. If the action returns HTTP 500 or another ambiguous
result, do not replay it. Re-fetch the chat state when it is observable;
otherwise report the outcome as indeterminate.

The Teams action bodies in this file are known deployed contracts; call them
directly. Use `get_schema` only for an undocumented action shape.

## Inspecting channel-message create properties

For "What properties can I set when creating a Teams channel message?", make
exactly one `get_schema` call for
`/teams/{teamId}/channels/{channelId}/messages` with
`operationType="create"`. Do not probe chat or update schemas.

Use that create schema as the source of truth for the answer. Lead with
user-supplied content fields such as `body`, `attachments`, `mentions`, and
other fields explicitly supported by the create payload. Do not present
system-generated or read-only resource fields as settable; this includes
identifiers, timestamps, sender and location metadata, reactions, replies,
hosted contents, and message history. If the returned schema exposes a broad
resource model without reliable writability annotations, state that limitation
instead of claiming every exposed property can be supplied on create.
