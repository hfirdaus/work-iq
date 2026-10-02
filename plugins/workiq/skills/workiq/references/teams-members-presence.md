# Teams members, tags, and presence

Use **Finding Teams targets** in `references/teams-routing.md` to resolve the team, channel, or
person. Use this file for membership identity, role changes, team tags, and
presence.

Removing members and deleting tags or tag members affect other people. Act only
on the exact resolved membership, tag, or tag-member ID; never on a similar
name.

## Paths

| Operation | Tool | Path |
| --- | --- | --- |
| List channel members | `fetch` | `/teams/{teamId}/channels/{channelId}/members` |
| Update a channel member | `update_entity` | `/teams/{teamId}/channels/{channelId}/members/{membershipId}` |
| List team tags | `fetch` | `/teams/{teamId}/tags` |
| Create a team tag | `create_entity` | parentUrl `/teams/{teamId}/tags` |
| Read one team tag | `fetch` | `/teams/{teamId}/tags/{tagId}` |
| List team-tag members | `fetch` | `/teams/{teamId}/tags/{tagId}/members` |
| Add a team-tag member | `create_entity` | parentUrl `/teams/{teamId}/tags/{tagId}/members` |
| Read presence | `fetch` | `/me/presence`, `/users/{id-or-UPN}/presence` (no `$select`) |
| Set my presence | `do_action` | `/me/presence/setUserPreferredPresence` |
| Reset presence to automatic | `do_action` | `/me/presence/clearUserPreferredPresence` with `{}` |
| Set a status message | `do_action` | `/me/presence/setStatusMessage` (body below) |
| List team members / owners | `fetch` | `/teams/{teamId}/members` |
| Change a team member's role | `update_entity` | `/teams/{teamId}/members/{membershipId}` |
| Add people to a team | `do_action` | `/teams/{teamId}/members/add` (body below) |
| Remove a team member | `delete_entity` | `/teams/{teamId}/members/{membershipId}` |
| Delete a team tag | `delete_entity` | `/teams/{teamId}/tags/{tagId}` |
| Remove a person from a tag | `delete_entity` | `/teams/{teamId}/tags/{tagId}/members/{tagMemberId}` |

## Members, owners, and mentions

Membership collections expose the base `conversationMember` shape. Do not
request `email`, `userId`, or `tenantId` as selected member fields.

- When only owners are needed, filter the scoped membership collection with
  `roles/any(...)` rather than sampling every member first.
- For a read-only member list, use the returned `displayName` and identity data.
- For a membership mutation or a true `@mention`, first identify the person in
  the scoped team or channel membership, then resolve the exact person through
  `/users/{exact-UPN}?$select=id,displayName,userPrincipalName,mail` (or the
  exact-name `/users?$filter=...` lookup in **Finding Teams targets**) to
  obtain the directory user ID required by the write payload.
- If a directory ID cannot be resolved safely, use plain-text addressing when
  acceptable or stop and report the limitation. Do not invent a derived member
  field.

### Listing members of a named channel

Follow the channel-members row in `SKILL.md`.

## Updating a channel member role

Use the membership ID returned by
`/teams/{teamId}/channels/{channelId}/members` in
`/teams/{teamId}/channels/{channelId}/members/{membershipId}`. Do not put the
directory user ID in the membership path. To promote a member to owner, make
one `update_entity` with:

```json
{"@odata.type":"#microsoft.graph.aadUserConversationMember","roles":["owner"]}
```

Always include `@odata.type`; do not first try `roles` alone. Do not remove and
re-add the member to change their role.

## Team membership

Team memberships work like channel memberships: read `/teams/{teamId}/members`
once, select the person's membership `id` (not their directory user ID), and:

- **Promote/demote:** `update_entity` `/teams/{teamId}/members/{membershipId}`
  with `{"@odata.type":"#microsoft.graph.aadUserConversationMember","roles":["owner"]}`
  (`"roles":[]` for a regular member).
- **Remove:** `delete_entity` `/teams/{teamId}/members/{membershipId}`.
- **Add one or more people:** resolve each directory user, then one `do_action`
  `/teams/{teamId}/members/add` with
  `{"values":[{"@odata.type":"microsoft.graph.aadUserConversationMember","roles":[],"user@odata.bind":"https://graph.microsoft.com/v1.0/users('{userId}')"}]}`
  listing everyone. The response has one result per user; report each user's
  outcome and any `error`.

These are known contracts; skip `search_paths` and `get_schema`.

## Team tags

Resolve the exact team once. For an existing tag, read `/teams/{teamId}/tags`
and match its display name exactly. Use query-free tag reads, and skip
`search_paths` and `get_schema` for these known routes and payloads.

| Operation | Request |
| --- | --- |
| Create a tag | `create_entity`, parentUrl `/teams/{teamId}/tags`, body `{"displayName":"{tagName}","members":[{"userId":"{directoryUserId}"}]}` |
| Add a tag member | `create_entity`, parentUrl `/teams/{teamId}/tags/{tagId}/members`, body `{"userId":"{directoryUserId}"}` |
| Read one tag | Use the list result when it already contains the requested ID, display name, and member count; otherwise `fetch` `/teams/{teamId}/tags/{tagId}`. Do not invent absent fields. |

Resolve each person's directory ID from the team roster or `/users/{exact-UPN}`.
Use directory user IDs, not membership IDs; omit member `displayName`, `roles`,
and `user@odata.bind` from tag payloads.

Before adding a member, read `/teams/{teamId}/tags/{tagId}/members` and compare
`userId` with the resolved directory ID. If already present, report that
without creating a duplicate. Otherwise add once, preserving existing members;
do not recreate the tag or remove and re-add its roster. Confirm from the
successful response and include the returned tag or member ID.

To remove a person from a tag, read `/teams/{teamId}/tags/{tagId}/members`,
match the person by `userId` or `displayName`, and delete
`/teams/{teamId}/tags/{tagId}/members/{tagMemberId}` using that entry's `id`
(the tag-member ID, never the user ID). Delete a tag with `delete_entity` on
`/teams/{teamId}/tags/{tagId}`. Renaming a tag is not available through WorkIQ
(the PATCH route is not exposed); say so, and do not delete and recreate the tag
unless the user explicitly asks for that.

## Presence

- "Set my presence to Busy/Away/DoNotDisturb" → `do_action` on
  `/me/presence/setUserPreferredPresence` with
  `{"availability": "Busy", "activity": "Busy", "expirationDuration": "PT1H"}`.
  This is the user-preferred presence and the right route for user requests.
- `/me/presence/setPresence` is the **application session** variant and requires a `sessionId` —
  only use it if you have one. If a presence write has an ambiguous result, do
  not replay it; fetch the current presence when possible and otherwise report
  the outcome as indeterminate. Do not cycle through alternate presence
  endpoints.
- Presence reads reject `$select`; fetch `/me/presence` or
  `/users/{id-or-UPN}/presence` directly and batch multiple people in one
  `fetch` call when useful.
- Preferred presence only shows while the user has an active Teams session; a
  read that still says `Offline` after a successful set is expected. Report the
  successful set and its expiry rather than claiming failure.
- "Reset my status / back to automatic" → `do_action`
  `/me/presence/clearUserPreferredPresence` with `{}`. Never set `Available`
  instead.
- Status message → `do_action` `/me/presence/setStatusMessage` with
  `{"statusMessage":{"message":{"content":"{text}","contentType":"text"},"expiryDateTime":{"dateTime":"{localDateTime}","timeZone":"{windowsOrIanaZone}"}}}`.
  Omit `expiryDateTime` when no end time is given; do not add `@odata.type`
  annotations. Clear it by sending an empty `content`.
- Work location for today and presence location: see
  `references/teams-lifecycle-settings.md`.
