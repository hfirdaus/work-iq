# Teams lifecycle, channels, and work settings

Use with `references/teams-routing.md` (lookups, **Member body**, write
outcomes). This file covers team details and settings, archiving, channels,
cross-team inventory, and work hours and location.

## Teams

| Operation | Tool | Path and body |
| --- | --- | --- |
| Description, visibility, archived state | `fetch` | `/teams/{teamId}?$select=id,displayName,description,visibility,isArchived` |
| What members may do (channels, apps, tabs, messages) | `fetch` | `/teams/{teamId}?$select=memberSettings,messagingSettings` (both groups in one fetch) |
| Change description (owner) | `update_entity` | `/teams/{teamId}` with only `{"description":"{text}"}` |
| Archive (owner) | `do_action` | `/teams/{teamId}/archive` with `{"shouldSetSpoSiteReadOnlyForMembers":false}` (or `{}`); send `true` only when the user asks to make the site read-only |
| Unarchive (owner) | `do_action` | `/teams/{teamId}/unarchive` with `{}` |
| Organization-wide Teams settings | `fetch` | `/teamwork`, at most once; it needs admin consent, so on access denied say the setting needs a Teams admin and never guess a region |

Answer settings questions from the returned booleans
(`memberSettings.allowCreateUpdateChannels`, `allowCreatePrivateChannels`,
`allowDeleteChannels`; `messagingSettings.allowUserEditMessages`,
`allowUserDeleteMessages`).

## Channels

| Operation | Tool | Path and body |
| --- | --- | --- |
| Every channel with membership type | `fetch` | `/teams/{teamId}/allChannels?$select=id,displayName,membershipType` |
| Create a private channel | `create_entity` | parentUrl `/teams/{teamId}/channels`, `{"displayName":"{name}","membershipType":"private","members":[...]}` with a **member body** per person: the caller `["owner"]`, others `[]` |
| Create a shared channel | `create_entity` | parentUrl `/teams/{teamId}/channels`, `{"displayName":"{name}","membershipType":"shared"}` |
| Archive or unarchive a channel | `do_action` | `/teams/{teamId}/channels/{channelId}/archive` or `/unarchive` with `{}` |

- **Private channel members can be dropped:** a delegated create can return 201
  with only the caller in the roster. If the user asked for others, read the
  new channel's `/members` once and add each missing person (see
  `references/teams-members-presence.md`). Report the roster as read, not as
  requested.
- **Shared channel create returns 202 with no ID:** fetch
  `/teams/{teamId}/channels?$select=id,displayName,membershipType` once (retry
  at most once if it is not listed yet), take the exact new channel, then add
  any extra owner. Report a policy block honestly; never substitute a standard
  or private channel.
- When a channel email address is requested, report the returned `email`; an
  empty value means no address is provisioned.

## Inventory across teams

- **Channels and owners across my teams:** fetch `/me/joinedTeams` once, then
  batch each team's `/teams/{teamId}/allChannels` (plus
  `/teams/{teamId}/installedApps?$expand=teamsAppDefinition` when apps are
  asked for) in one `fetch`. Standard channels inherit team owners: read
  `/teams/{teamId}/members?$filter=roles/any(r:r eq 'owner')` once per team,
  and read channel members only for private or shared channels. Report a team
  that returns 403 or another 4xx as inaccessible and continue.
- **Every team I can reach, including through shared channels:** fetch
  `/me?$select=id` with `/me/joinedTeams?$select=id,displayName`, then
  `/users/{id}/teamwork/associatedTeams` (the `/me/teamwork/...` form is
  denied). Teams in `associatedTeams` but not `joinedTeams` are
  shared-channel-only access.

## Work hours and work location

| Request | Tool | Path and body |
| --- | --- | --- |
| Working hours, days, default location | `fetch` | `/me/settings/workHoursAndLocations/recurrences` (the base `/me/settings/workHoursAndLocations` does not list hours) |
| Set **today's** location | `do_action` | `/me/settings/workHoursAndLocations/occurrences/setCurrentLocation` with `{"workLocationType":"office"}` (`remote` for home; valid values are `office`, `remote`, `timeOff`, `unspecified`) |
| Change a weekday **going forward** | `update_entity` | `/me/settings/workHoursAndLocations/recurrences/{recurrenceId}` (body below) |
| Show a location in presence | `do_action` | `/me/presence/setManualLocation` with `{"workLocationType":"remote"}` (`office` for the office); clear with `/me/presence/clearLocation` and `{}` |

"Show me as …" is a presence display request (`setManualLocation`); use
`setCurrentLocation` only when the user asks to set or change their work
location or work plan. Do not create work-plan occurrences for "today".

Recurrences hold one weekly entry per workday with `start`/`end` in the user's
time zone and `workLocationType` (`unspecified` means no default location). To
change a weekday going forward, update that day's existing recurrence; creating
another one fails as an overlapping segment. The PATCH must include
`workLocationType` plus `start`, `end`, and `recurrence` (pattern and range)
copied from the fetched recurrence; a body with only `workLocationType` is
rejected. The service returns the updated recurrence under a new ID:

```json
{"workLocationType":"remote","start":{"dateTime":"{existingStart}","timeZone":"{tz}"},"end":{"dateTime":"{existingEnd}","timeZone":"{tz}"},"recurrence":{"pattern":{"type":"weekly","interval":1,"daysOfWeek":["friday"]},"range":{"type":"noEnd","startDate":"{existingStartDate}","recurrenceTimeZone":"{tz}"}}}
```
