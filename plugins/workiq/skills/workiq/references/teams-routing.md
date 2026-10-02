# Teams routing and stopping rules

Use this file for target lookup, deployed query exceptions, bounded reads, and
error handling.

## Read paths

| Operation | Tool | Path |
| --- | --- | --- |
| List my chats | `fetch` | `/me/chats?$expand=members` |
| List members of a named group chat | `fetch` | `/me/chats?$filter=topic%20eq%20%27{odataEscapedAndUrlEncodedExactTopic}%27&$expand=members&$top=50` (answer from the expanded `members`; never `ask`) |
| List messages in a chat | `fetch` | `/chats/{chatId}/messages` |
| List my teams | `fetch` | `/me/joinedTeams?$select=id,displayName` |
| List a team's channels | `fetch` | `/teams/{teamId}/channels?$select=id,displayName` |
| Read channel details | `fetch` | `/teams/{teamId}/channels/{channelId}?$select=id,displayName,description,membershipType` |
| List channel messages | `fetch` | `/teams/{teamId}/channels/{channelId}/messages` |
| Read channel-thread replies | `fetch` | `/teams/{teamId}/channels/{channelId}/messages/{messageId}/replies` |
| Channel-message delta ("what's new since…") | `call_function` | `/teams/{teamId}/channels/{channelId}/messages/delta` |

Query options for these paths follow **Deployed Teams query exceptions** below.

## Finding Teams targets

Every Teams task starts from these lookups. Apply them to all Teams reads and
mutations:

- Match names and message text exactly. Never act on a partial, similar, or
  semantic match.
- Do not use `ask` to find a chat, channel, or message that will be changed.
  If the exact target is not found, report it as not found.
- Omit `$top` on `/me/joinedTeams`, `/chats/{chatId}/messages`, and
  `/teams/{teamId}/channels/{channelId}/messages`.

### Finding a channel

1. Fetch exactly `/me/joinedTeams?$select=id,displayName` and select the exact
   team name. Do not add `$top`; the deployed endpoint rejects it.
2. Fetch `/teams/{teamId}/channels?$select=id,displayName` and select the exact
   channel name. Do not choose the first similar channel name.

- **General (primary) channel:** fetch `/teams/{teamId}/primaryChannel`
  directly.
- **Shared channels shared into a team:** fetch
  `/teams/{teamId}/incomingChannels` once without `$select` (an empty list
  means none).

### Finding a chat

Pick the lookup that matches how the user named the chat.

**By person (1:1 chat).** Microsoft Graph permits only one one-on-one chat for
a pair of users. If it already exists, this call returns that existing chat
instead of creating a duplicate. Require a non-empty returned chat ID and
`chatType == "oneOnOne"`. Do not enumerate `/me/chats` for a named person.

1. Resolve the signed-in user and a verified directory-user counterpart:
   - When the user supplied an email address or UPN, fetch `/me?$select=id` and
     `/users/{urlEncodedUserPrincipalName}?$select=id,displayName,mail,userPrincipalName`.
   - Otherwise, fetch `/me?$select=id` and
     `/users?$filter=displayName%20eq%20%27{odataEscapedAndUrlEncodedExactDisplayName}%27&$select=id,displayName,mail,userPrincipalName&$top=10`.
2. Require exactly one returned directory user whose `displayName` exactly
   matches the requested person. If no user or multiple users match, ask for
   an email address or UPN instead of guessing. Do not use `/me/people`;
   People results can be fuzzy or represent contacts rather than directory
   users.
3. Call `create_entity` with `parentUrl="/chats"` and exactly these two members,
   using only the returned directory-user `id` for `{counterpartUserId}`:

```json
{
  "chatType": "oneOnOne",
  "members": [
    {
      "@odata.type": "#microsoft.graph.aadUserConversationMember",
      "roles": ["owner"],
      "user@odata.bind": "https://graph.microsoft.com/v1.0/users('{signedInUserId}')"
    },
    {
      "@odata.type": "#microsoft.graph.aadUserConversationMember",
      "roles": ["owner"],
      "user@odata.bind": "https://graph.microsoft.com/v1.0/users('{counterpartUserId}')"
    }
  ]
}
```

**By topic (group chat).** In the initial `fetch` call, request
`/me?$select=id` and
`/me/chats?$filter=topic%20eq%20%27{odataEscapedAndUrlEncodedExactTopic}%27&$expand=members&$top=50`,
and require an exact `topic` match. If the response includes
`@odata.nextLink`, follow the global pagination and partial-result guidance in
`references/fetch-work-iq.md`.

If the exact-topic filter returns no chats, do not fall back to `ask`; report
the chat as not found.

**Your member identity in the chat.** `hideForUser`, `markChatReadForUser`,
and `markChatUnreadForUser` need the signed-in member whose `userId` equals
`{signedInUserId}`. If the chat lookup already returned members (as the topic
lookup does), use them. Do not fetch `/chats/{chatId}/members` again.
Otherwise, fetch exactly `/chats/{chatId}/members`. The URL must end at
`/members`; do not append any query string, including `$select`, `$expand`, or
`$top`. `userId` and
`tenantId` are returned by the unfiltered response but are not selectable
`conversationMember` properties. Put that member's `userId` in
`teamworkUserIdentity.id` and use the same member's returned `tenantId`. Never
use the conversation member's opaque `id` value (often beginning with `MCMj`);
Graph can interpret it as another user and return HTTP 403.

### Finding a message

First find the channel or chat, then fetch its messages and match the complete
message text exactly:

- Channel: `/teams/{teamId}/channels/{channelId}/messages?$select=id,createdDateTime,body`
- Chat: `/chats/{chatId}/messages?$select=id,createdDateTime,body`

Use the matching message's `id` in the follow-up call.

When searching or summarizing messages rather than matching one exact text,
add `from,subject` to that `$select` so senders are visible. `attachments` and
`reactions` cannot be selected (the request fails); when shared files or
reactions matter, fetch the collection **without** `$select`. For file
attachments, follow **Channel message file attachments** in
`references/teams-apps-files-content.md`.

## Listing my chats

Follow the list-chats row in `SKILL.md`.

## Which chats are unread

Fetch `/me/chats?$expand=lastMessagePreview` once and compare each chat's
`viewpoint.lastMessageReadDateTime` with the preview's `createdDateTime`.
A preview whose `messageType` is `systemEventMessage` (members added, chat
renamed) is not unread conversation: for those candidates, fetch
`/chats/{chatId}/messages?$select=id,createdDateTime,messageType,from,body`
(batch the candidate chats in one `fetch`) and count only `messageType`
`message` items newer than the read time and not sent by the user. Report each
unread chat with the unread sender and gist, and say which chats are caught up.

## Summarizing a named chat or channel

When the user names one chat or channel ("summarize decisions in my Release
readiness chat"), resolve it with **Finding Teams targets** and fetch its
messages directly (`/chats/{chatId}/messages` or the channel `messages` plus
`/replies` for threads with replies). Filter the requested date window locally.
Do not use `ask` for a single named container; reserve `ask` for open-ended
questions across unknown sources.

## Pinned chat messages

Fetch `/chats/{chatId}/pinnedMessages?$expand=message` (no `$select`; nested
`$select` inside the expansion is rejected) and report each pinned message's
sender, time, and text. An empty list or a 404 "pinnedItems was not found"
means nothing is pinned.

## Who reacted to a message

Reactions are returned on the message itself: read the channel or chat message
collection (or the single message) **without `$select`** — `reactions` cannot
be selected and such requests fail — and report each `reactions[]` entry's
`reactionType` (Unicode emoji or name) and `user.user.id`/`displayName`. Map
user IDs to names from the team or chat roster only when the reaction omits a
display name. There is no separate reactions endpoint.

## Cross-cutting route discipline

- **Target lock:** Once an exact team, channel, chat, message, thread, or meeting
  is confirmed, retain its canonical ID for the rest of the task. Do not inspect
  another same-named container unless the selected target is explicitly
  disproven.
- **Evidence sufficiency:** When a successful collection response already
  contains every requested ID, body, sender, timestamp, or metadata field, use
  that evidence directly. Do not issue an item-detail fetch solely for
  enrichment.
- **Batch independent resolution:** When identity and container lookups do not
  depend on each other, send them in one `fetch` call with multiple
  `entityUrls`, then reuse the returned IDs.
- **Chain mutation outputs:** Use the ID returned by `create_entity` directly in
  the next action. Do not rediscover the new entity or inspect an action schema
  before using that returned ID.

## Deployed Teams query exceptions

The generic `fetch` advice to add `$select`, `$top`, `$filter`, and `$orderby`
does not apply uniformly to Teams. These are deployed WorkIQ constraints, not
claims about every Microsoft Graph environment. When this table conflicts with
generic query guidance, this table wins.

| Path | Supported approach | Do not send |
| --- | --- | --- |
| `/me/joinedTeams` | `$select=id,displayName` | `$top` |
| `/teams/{teamId}/channels` | `$select=id,displayName` | `$top` |
| `/teams/{teamId}/members`, `/teams/{teamId}/channels/{channelId}/members` | Read base `conversationMember` fields; when only owners are needed, filter with `roles/any(...)` | `$top`; `email`, `userId`, or `tenantId` in `$select` |
| `/chats/{chatId}/members` | Fetch the unfiltered collection | any query string |
| `/teams/{teamId}/tags`, `/teams/{teamId}/tags/{tagId}/members` | Resolve the team, then read the base collection | `$select`, `$top`, `$filter`, `$orderby`; `search_paths` |
| `/me/chats` | `$expand=members` without nested projection; exact-topic `$filter` as in **Finding a chat** | `$orderby=lastUpdatedDateTime`; nested `$select` inside `$expand=members(...)` |
| `/chats/{chatId}/messages` | Fetch the collection and filter or sort locally | `$top`; created-date filters |
| `/teams/{teamId}/channels/{channelId}/messages` | Fetch one page and filter locally | `$top`; `$filter` (including `createdDateTime` ranges); `$orderby`; `$skiptoken` |
| `/chats/{chatId}/tabs` | Fetch the collection without a count limit | `$top` |
| `/me/teamwork/installedApps`, `/users/{id}/teamwork/installedApps` | Use the expanded collection directly | `$top` |
| `/me/presence`, `/users/{id-or-UPN}/presence` | Fetch the presence resource directly | `$select` |
| `getAllTranscripts(...)` | Call the function without projection and select the matching transcript locally | `$select` |

Do not retry rejected query variants after a 400; use the supported approach
and filter locally.

### Creation time versus modification time

Microsoft Graph documents `lastModifiedDateTime` filtering for
[chat messages](https://learn.microsoft.com/en-us/graph/api/chat-list-messages?view=graph-rest-1.0)
only together with `$orderby=lastModifiedDateTime desc` on the same request.
That does not establish support on
[channel messages](https://learn.microsoft.com/en-us/graph/api/channel-list-messages?view=graph-rest-1.0).

`lastModifiedDateTime` is not a substitute for `createdDateTime`: edits can move
older messages into a newer modification window. For messages sent during a
date range, filter `createdDateTime` locally and report partial coverage unless
the retrieved evidence establishes completeness.

## Bounded reads

- Start with minimal fields. Resolve IDs, names, timestamps, and URLs before
  requesting message bodies, meeting details, app definitions, or other rich
  fields.
- For catch-up or cross-container reads, resolve candidate container IDs first,
  then inspect at most the two highest-confidence chats or channels and fetch
  each selected message collection once. Never enumerate every accessible chat
  or channel as a fallback; if two candidates are insufficient, return a
  clearly labelled partial result.
- Fetch replies only for parent messages already selected as relevant. Stop
  once the requested decision, rationale, owners, and follow-ups are grounded.
- For activity sampling, start with `createdDateTime,subject,summary`; fetch
  `body` only for one selected activity or message.
- Before relying on semantic attribution, verify the sender from the selected
  collection result. Fetch the item directly only when the collection omitted
  the required identity field.
- If `ask` results contain a malformed chat ID, extract the canonical
  `19:...@thread.v2` ID from the cited Teams URL and verify that exact chat once.
- When paging a message list, fetch a page or two. Do not follow
  `@odata.nextLink` for dozens of pages, and do not replay or rewrite a rejected
  `$skiptoken`. Answer from the retrieved pages and state that the result is
  partial.

## Error-specific stopping rules

| Error | Required response | Do not |
| --- | --- | --- |
| 404 after selecting a message from a collection | Verify the container and ID from the already-read collection | Repeatedly retry the item route |
| Unsupported query option 400 | Use the documented base collection and filter locally | Probe query variants |
| Transcript-policy 403 (`GraphAccessToTranscriptsDisabled`) | Stop and report the policy | Try alternate IDs or surfaces |
| Activity-notification authorization 403 | Stop and report app authorization | Try alternate user or team IDs |
| Mutation payload 400 | Use the exact documented recipe | Guess fields or repeat schema calls |

## Resolve-then-act (do not loop)

1. Resolve the chat or team/channel with **Finding a chat** or **Finding a
   channel**.
2. If the target is not found, report "not found". For read-only synthesis you
   may try one `ask`; never use `ask` to find a mutation target.
3. Perform the requested mutation directly once you have the IDs — posting,
   replying, reacting, or editing is the goal, not enumerating message history.
4. After a successful mutation, report the result once; do not repeat the
   final message or action summary in multiple formats.
