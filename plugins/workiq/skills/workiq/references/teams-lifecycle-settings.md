# Teams lifecycle, channels, and work settings

Use **Finding Teams targets** in `references/teams-routing.md` to resolve the team or channel. The
routes and bodies below are known deployed contracts: call them directly
without `search_paths` or `get_schema`.

Archiving, description changes, and channel-file deletes affect everyone in the
team. Act only on the exact resolved team, channel, or file; never on a
similar name.

## Team details and settings

| Question | Call |
| --- | --- |
| Description, visibility, archived state | `fetch` `/teams/{teamId}?$select=id,displayName,description,visibility,isArchived` |
| What members may do (create/delete channels, apps, tabs) | `fetch` `/teams/{teamId}?$select=memberSettings,messagingSettings` |

Answer settings questions from the returned booleans
(`memberSettings.allowCreateUpdateChannels`, `allowCreatePrivateChannels`,
`allowDeleteChannels`; `messagingSettings.allowUserEditMessages`,
`allowUserDeleteMessages`). Read both settings groups in **one** fetch.

## Team writes (owner only)

| Operation | Tool | Request |
| --- | --- | --- |
| Change description | `update_entity` | `/teams/{teamId}` with only `{"description":"{text}"}` |
| Archive | `do_action` | `/teams/{teamId}/archive` with `{"shouldSetSpoSiteReadOnlyForMembers":false}` (or `{}`); send `true` only when the user asks to make the site read-only |
| Unarchive | `do_action` | `/teams/{teamId}/unarchive` with `{}` |

Archive and unarchive return **HTTP 202 Accepted**: say the request was
accepted and finishes asynchronously; do not claim it is already archived
unless a later read shows `isArchived`. Creating or cloning teams and
installing team apps are not available through WorkIQ; say so instead of
improvising.

## Channels

| Operation | Tool | Request |
| --- | --- | --- |
| A team's General (primary) channel | `fetch` | `/teams/{teamId}/primaryChannel` — do not filter `/channels` by name |
| Shared channels shared *into* a team | `fetch` | `/teams/{teamId}/incomingChannels` (empty `value` means none) |
| Every channel incl. membership type | `fetch` | `/teams/{teamId}/allChannels?$select=id,displayName,membershipType` |
| Create a private channel with members | `create_entity` | parentUrl `/teams/{teamId}/channels`, body below |
| Create a shared channel | `create_entity` | parentUrl `/teams/{teamId}/channels`, `{"displayName":"{name}","membershipType":"shared"}` |
| Add a channel member/owner | `create_entity` | parentUrl `/teams/{teamId}/channels/{channelId}/members`, `{"@odata.type":"#microsoft.graph.aadUserConversationMember","roles":["owner"],"user@odata.bind":"https://graph.microsoft.com/v1.0/users('{userId}')"}` |
| Archive / unarchive a channel | `do_action` | `/teams/{teamId}/channels/{channelId}/archive` or `/unarchive` with `{}` (HTTP 202) |

Private channel body (one call; the caller is an owner, others are members):

```json
{"displayName":"{name}","membershipType":"private","members":[{"@odata.type":"#microsoft.graph.aadUserConversationMember","roles":["owner"],"user@odata.bind":"https://graph.microsoft.com/v1.0/users('{signedInUserId}')"},{"@odata.type":"#microsoft.graph.aadUserConversationMember","roles":[],"user@odata.bind":"https://graph.microsoft.com/v1.0/users('{memberUserIdOrUpn}')"}]}
```

A delegated private-channel create can return 201 with only the caller in the
roster even when `members[]` listed others. If the user asked for other
members, read `/teams/{teamId}/channels/{channelId}/members` once and add each
missing person with the members call above (`"roles":[]` for a member); skip
`get_schema` for these known bodies.

A shared-channel create returns 202 with no ID. Fetch
`/teams/{teamId}/channels?$select=id,displayName,membershipType` once (retry at
most once if it is not listed yet), take the exact new channel, then add any
extra owner with the members call above. Report a policy block honestly; never
substitute a standard or private channel.

Channel deletion is not available through WorkIQ. If asked to delete a channel,
say it cannot be deleted here and offer to archive it instead; do not archive
without the user's agreement.

When an email address is requested for a channel, report the returned `email`
value; an empty value means no address is provisioned.

## Inventory across teams

For "channels and owners across my teams", fetch `/me/joinedTeams` once, then
batch per-team `/teams/{teamId}/allChannels` (plus
`/teams/{teamId}/installedApps?$expand=teamsAppDefinition` when apps are asked
for) in one `fetch`. Standard channels inherit team owners: read
`/teams/{teamId}/members?$filter=roles/any(r:r eq 'owner')` once per team.
Read `/teams/{teamId}/channels/{channelId}/members` only for private or shared
channels. If a team or resource returns 403/4xx, report it as inaccessible
and continue; do not retry it or follow `$skiptoken` (rejected by the server)
on a failed page. One failing URL marks a whole batched `fetch` as failed, but
the error payload still carries every other URL's result: use those results
instead of re-fetching them.

## Organization-wide Teams settings

`/teamwork` (tenant Teams enablement and region) requires admin-consented
permissions and is denied for ordinary users. Try it at most once, then say the
setting cannot be read with the user's permissions and suggest asking a Teams
admin. Never guess a region.

For "every team I can reach, including through shared channels", fetch
`/me?$select=id` with `/me/joinedTeams?$select=id,displayName`, then
`/users/{id}/teamwork/associatedTeams` (the `/me/teamwork/...` form is denied).
Teams in associatedTeams but not joinedTeams are shared-channel-only access.

Not available through WorkIQ (say so once instead of probing): creating or
cloning teams, deleting channels, installing team apps, reading the app catalog,
uploading files, and creating sharing links. Channel-file deletes do work:
`delete_entity` on `/drives/{driveId}/items/{itemId}` from the channel's
`filesFolder` drive.

## Work hours and work location

| Question / request | Tool | Request |
| --- | --- | --- |
| Working hours, days, default location | `fetch` | `/me/settings/workHoursAndLocations/recurrences` (the base `/me/settings/workHoursAndLocations` resource does not list hours) |
| Set **today's** location | `do_action` | `/me/settings/workHoursAndLocations/occurrences/setCurrentLocation` with `{"workLocationType":"office"}` (`remote` for home; valid values are `office`, `remote`, `timeOff`, `unspecified`) |
| Change a weekday **going forward** | `update_entity` | `/me/settings/workHoursAndLocations/recurrences/{recurrenceId}` with `workLocationType` **plus the required `start`, `end`, and `recurrence`** copied from that fetched recurrence |
| Show a location in presence | `do_action` | `/me/presence/setManualLocation` with `{"workLocationType":"remote"}`; clear with `/me/presence/clearLocation` and `{}` |

Recurrences hold one weekly entry per workday with `start`/`end` in the
user's time zone and `workLocationType` (`unspecified` means no default
location). Fetch the collection without `$select` (`daysOfWeek` lives inside
`recurrence.pattern` and cannot be selected). To change a weekday going
forward, update that day's existing recurrence; creating another one fails as an
overlapping segment. `start`, `end`, and `recurrence` (pattern and range) are
required on this PATCH: a body with only `workLocationType` is rejected. Copy all
three from the fetched recurrence into the first PATCH; do not send a partial
body and retry. The service returns the updated recurrence under a new ID:

```json
{"workLocationType":"remote","start":{"dateTime":"{existingStart}","timeZone":"{tz}"},"end":{"dateTime":"{existingEnd}","timeZone":"{tz}"},"recurrence":{"pattern":{"type":"weekly","interval":1,"daysOfWeek":["friday"]},"range":{"type":"noEnd","startDate":"{existingStartDate}","recurrenceTimeZone":"{tz}"}}}
```

Do not create work-plan occurrences for "today"; use `setCurrentLocation`.
