# Teams apps, tabs, files, and hosted content

Use **Finding Teams targets** in `references/teams-routing.md` to resolve the chat, team, or
channel. Use this file for app/file discovery and blob retrieval.

## Paths

| Operation | Tool | Path |
| --- | --- | --- |
| List chat tabs | `fetch` | `/chats/{chatId}/tabs?$expand=teamsApp` |
| List channel tabs | `fetch` | `/teams/{teamId}/channels/{channelId}/tabs?$expand=teamsApp` |
| List apps installed in a chat | `fetch` | `/chats/{chatId}/installedApps?$expand=teamsAppDefinition` |
| List apps installed in a team | `fetch` | `/teams/{teamId}/installedApps?$expand=teamsAppDefinition` |
| List apps installed for me or a user | `fetch` | `/me/teamwork/installedApps?$expand=teamsAppDefinition`, `/users/{id}/teamwork/installedApps?$expand=teamsAppDefinition` (no `$top`) |
| Channel files folder | `fetch` | `/teams/{teamId}/channels/{channelId}/filesFolder` |
| List message hosted content | `fetch` | `/chats/{chatId}/messages/{messageId}/hostedContents` |
| Download inline hosted content | `fetch_blob` | `/chats/{chatId}/messages/{messageId}/hostedContents/{hostedContentId}/$value` |

## Apps and tabs

- Installed-app results cannot distinguish a personally installed app from one
  assigned by policy; do not claim which applies. Report app display names
  first, and include IDs, versions, or definitions only for specifically
  requested apps.
- Use the canonical expanded collections directly; do not call `search_paths`,
  `get_schema`, or add `$top`.
- The Teams app catalog (`/appCatalogs/teamsApps`) and installing apps are not
  available through WorkIQ; say so instead of probing.

### Adding, renaming, and removing tabs

These are known contracts; skip `search_paths` and `get_schema`.

- **Add a website tab:** `create_entity` with parentUrl
  `/teams/{teamId}/channels/{channelId}/tabs` (or `/chats/{chatId}/tabs`) and
  `{"displayName":"{name}","teamsApp@odata.bind":"https://graph.microsoft.com/v1.0/appCatalogs/teamsApps/com.microsoft.teamspace.tab.web","configuration":{"contentUrl":"{url}","websiteUrl":"{url}"}}`.
  The Website app ID is fixed; do not look it up.
- **Rename:** list the tabs, match the exact `displayName`, then
  `update_entity` `.../tabs/{tabId}` with only `{"displayName":"{newName}"}`.
- **Remove:** list the tabs, match exactly, then `delete_entity`
  `.../tabs/{tabId}`.

## Channel files

When discovering channel files, prioritize exact Teams-domain results and do
not follow unrelated mail, contact, or generic folder suggestions.

## Channel message file attachments

A shared file appears in the message's `attachments` array with
`contentType` `reference`, its file `name`, and a SharePoint `contentUrl`.
Always report each attachment's file name alongside the message. Pick the
starting point from the request:

- **A file shared in a post or message** ("the file Maya shared in General"):
  read the channel messages and their attachments first.
- **A file stored in a channel's Files** ("find the Q3 plan doc in the Work IQ
  team"): use the channel files folder route below first. Folders can be empty
  even when files were shared in posts, so if the file is in no channel folder,
  read the channel messages' attachments.

When the user asks what a message or topic contains, read the file
rather than describing it only as "an attachment":

1. Encode the attachment `contentUrl` as a sharing ID:
   `u!` + base64url(contentUrl) with `=` padding removed, `/` → `_`,
   `+` → `-`.
2. Call `fetch_blob` once on `/shares/{sharingId}/driveItem/content` (or fetch
   `/shares/{sharingId}/driveItem` first when you need its `webUrl` or IDs).
3. Summarize the relevant facts from the file.

Fall back to the channel `filesFolder` route below to download an attachment
only when the message has no `contentUrl`. To share an attached file in another channel, post the
attachment's `contentUrl` (or the driveItem `webUrl`) as a link in the new
message body.

Channel files folder route:

1. Fetch `/teams/{teamId}/channels/{channelId}/filesFolder` and keep
   `parentReference.driveId` and the folder `id`.
2. Fetch `/drives/{driveId}/items/{folderId}/children?$select=id,name,file,webUrl`
   and select the item whose `name` exactly matches the attachment name.
3. Call `fetch_blob` once on `/drives/{driveId}/items/{itemId}/content`, then
   summarize the relevant facts from the file.

If the file cannot be found or the download fails, say so and still give the
file name and link; do not guess its contents. Creating sharing links and
uploading files are not available through WorkIQ.

## Latest inline image from a named chat and sender

1. Resolve the exact chat with **Finding Teams targets**.
2. Fetch `/chats/{chatId}/messages` once and read sender, timestamp, body,
   attachments, and hosted-content metadata.
3. Filter locally to the requested sender, select the latest matching message,
   and select its hosted-content ID.
4. Call `fetch_blob` once on
   `/chats/{chatId}/messages/{messageId}/hostedContents/{hostedContentId}/$value`.

Never fetch the hostedContents collection for every message, search unrelated
teams or chats, or call `ask` after the exact chat has been resolved. Treat the
blob response's MIME metadata as authoritative when the hosted-content metadata
is null.

In the answer, give the sender, timestamp, and message text, and briefly
describe what the image shows when its content is visible to you.
