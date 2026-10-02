# Teams meetings

Use this file for meeting resolution, transcripts, AI insights, and
transcript-policy stops.

## Paths

| Operation | Tool | Path |
| --- | --- | --- |
| Resolve an online meeting from an event | `fetch` | `/me/onlineMeetings?$filter=JoinWebUrl%20eq%20%27{urlEncodedJoinUrl}%27` |
| List meeting transcripts | `fetch` | `/me/onlineMeetings/{meetingId}/transcripts` |
| Download transcript content | `fetch_blob` | `/me/onlineMeetings/{meetingId}/transcripts/{transcriptId}/content` with `format: "text/vtt"` |
| Read meeting AI insights | `fetch` | `/copilot/users/{userId}/onlineMeetings/{meetingId}/aiInsights` |

## Resolving a meeting

Start with minimal fields. Fetch a bounded `/me/calendarView` with
`$select=id,subject,start,end,onlineMeeting` and select exactly one
occurrence before requesting rich fields such as `body`, `attendees`, or
`organizer`. If calendarView returns `onlineMeeting: null` for an event that
is a Teams meeting, read that event once through `/me/events/{id}` (or
`/me/events?$filter=startswith(subject,%27{odataEscapedAndUrlEncodedTitle}%27)`) with
`$select=subject,onlineMeeting`.

| Request | Route |
| --- | --- |
| "What Teams meetings do I have today/this week?" | One `fetch` of `/me/calendarView?startDateTime=...&endDateTime=...&$select=subject,start,end,isOnlineMeeting,onlineMeeting,organizer`; list only `isOnlineMeeting` events with their `onlineMeeting.joinUrl`, and mention non-Teams events only as excluded |
| "What's the join link for {meeting}?" | `fetch` `/me/events?$filter=startswith(subject,%27{odataEscapedAndUrlEncodedTitle}%27)&$select=subject,start,end,organizer,onlineMeeting` (or a bounded calendarView) and return `onlineMeeting.joinUrl` |
| "Who organized the meeting at this link?" | `fetch` `/me/onlineMeetings?$filter=JoinWebUrl%20eq%20%27{urlEncodedJoinUrl}%27`, URL-encoding the join URL as given; read `subject`, `startDateTime`, `endDateTime`, `participants.organizer` |
| Read or post in a meeting's chat | Resolve the chat by topic (`/me/chats?$filter=topic%20eq%20%27{odataEscapedAndUrlEncodedTitle}%27`, chatType `meeting`) or from the online meeting's `chatInfo.threadId`, then `/chats/{chatId}/messages` |

Schedule a Teams meeting with one `create_entity` on `/me/events` including
`"isOnlineMeeting":true,"onlineMeetingProvider":"teamsForBusiness"`, the
attendees, and `start`/`end` in the user's time zone; report the returned
`onlineMeeting.joinUrl`.

## Meeting transcripts

1. Resolve one calendar occurrence as above.
2. Read that occurrence's `onlineMeeting.joinUrl` and resolve the online meeting
   with `/me/onlineMeetings?$filter=JoinWebUrl%20eq%20%27{urlEncodedJoinUrl}%27`.
3. Fetch `/me/onlineMeetings/{meetingId}/transcripts`.
4. Select the matching transcript and call `fetch_blob` on
   `/me/onlineMeetings/{meetingId}/transcripts/{transcriptId}/content` with
   `format: "text/vtt"`.

Do not call `ask`, `get_schema`, `getAllTranscripts`, AI insights, or meeting
chat endpoints before one calendar occurrence and one online-meeting ID are
resolved. A chat ID is never an online-meeting ID. If a required ID cannot be
resolved, stop and report the missing prerequisite rather than trying
alternate meeting or chat identifiers.

If `getAllTranscripts(...)` is used, call it without `$select` and select the
matching transcript locally.

## Meeting AI insights

After resolving the online-meeting ID, fetch
`/copilot/users/{userId}/onlineMeetings/{meetingId}/aiInsights`.

## Transcript-policy stop

If transcript access returns HTTP 403 with `GraphAccessToTranscriptsDisabled`,
stop and report the transcript policy. Do not try alternate IDs or surfaces.
If a meeting has no transcript or recap, say so rather than inferring its
outcome.
