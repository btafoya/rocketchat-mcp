# Rocket.Chat REST API Reference for `rocketchat-mcp`

This document is a local planning reference for expanding the
`rocketchat-mcp` MCP server. It combines the endpoints already wrapped by
`rocketchat.py`, high-value expansion candidates with concrete request/response
shapes, and a full category index linking to the official Rocket.Chat API docs.

Official documentation: https://developer.rocket.chat/apidocs

---

## Authentication

Every REST call requires these headers:

| Header | Value |
| --- | --- |
| `X-Auth-Token` | `authToken` (Personal Access Token or login token) |
| `X-User-Id` | `userId` |

The current server supports two initialization paths:

- **Personal Access Token** (recommended): set `ROCKETCHAT_USER_ID` and
  `ROCKETCHAT_AUTH_TOKEN`.
- **Username/password**: `POST /api/v1/login` returns `{ data: { authToken, userId } }`.

---

## Already wrapped endpoints

These endpoints are already used by tools in `rocketchat.py`.

| Tool | Method | Endpoint | Notes |
| --- | --- | --- | --- |
| `send_message_in_channel`, `send_direct_message` | POST | `chat.postMessage` | `channel` accepts room ID, name, or `@username` |
| `delete_message` | POST | `chat.delete` | Needs `roomId` + `msgId` |
| `get_channel_messages` | GET | `channels.messages`, `groups.messages`, `im.messages` | Probed and cached per room |
| `search_messages` | GET | `chat.search` | Requires resolved `roomId` |
| `get_unread` | GET | `subscriptions.get` | Fetches `update[]` array |
| `send_file` | POST | `im.create` then `rooms.upload/{roomId}` | For `@user` DMs |
| `download_attachment` | GET | `chat.getMessage` + `/file-upload/{fileId}/{fileName}` | Writes to temp dir |
| `list_all_rooms` | GET | `channels.list`, `groups.list`, `im.list` | Requires view permissions |
| `list_users` | GET | `users.list` | Requires admin or view permission |
| `get_user_info` | GET | `users.info` | Query by `username` |
| `create_channel` | POST | `channels.create` | |

---

## High-priority expansion candidates

These endpoints map cleanly to new MCP tools and are commonly needed in a chat
assistant context.

### Message operations

| Candidate tool | Method | Endpoint | Body / Query | Permissions / notes |
| --- | --- | --- | --- | --- |
| `update_message` | POST | `chat.update` | `roomId`, `msgId`, `text` | Edits own messages |
| `pin_message` | POST | `chat.pinMessage` | `messageId` | |
| `unpin_message` | POST | `chat.unpinMessage` | `messageId` | |
| `star_message` | POST | `chat.starMessage` | `messageId` | |
| `unstar_message` | POST | `chat.unstarMessage` | `messageId` | |
| `react_to_message` | POST | `chat.react` | `messageId`, `emoji` or `reaction`, optional `shouldReact` | Add/remove reaction |
| `get_thread_messages` | GET | `chat.getThreadMessages` | Query `tmid` (thread parent ID), `count`, `offset` | |
| `get_pinned_messages` | GET | `chat.getPinnedMessages` | Query `roomId` | |
| `get_starred_messages` | GET | `chat.getStarredMessages` | Query `roomId` | |
| `get_mentioned_messages` | GET | `chat.getMentionedMessages` | Query `roomId` | |

#### Example: `POST /api/v1/chat.update`

```json
{
  "roomId": "ROOM_ID",
  "msgId": "MSG_ID",
  "text": "Updated text"
}
```

Response fields include `_id`, `rid`, `msg`, `ts`, `u`, `_updatedAt`, `editedAt`,
`editedBy`.

#### Example: `POST /api/v1/chat.react`

```json
{
  "messageId": "MSG_ID",
  "emoji": "smile",
  "shouldReact": "true"
}
```

Response: `{ "success": true }`.

#### Example: `GET /api/v1/chat.getThreadMessages`

Query: `?tmid=THREAD_PARENT_ID&count=50`

Response: `{ "messages": [...], "count", "offset", "total", "success" }`.

### Room/channel/group management

| Candidate tool | Method | Endpoint | Body / Query | Permissions / notes |
| --- | --- | --- | --- | --- |
| `create_group` | POST | `groups.create` | `name`, optional `members` | Private channel |
| `archive_channel` | POST | `channels.archive` | `roomId` or `channel` | |
| `unarchive_channel` | POST | `channels.unarchive` | `roomId` or `channel` | |
| `archive_group` | POST | `groups.archive` | `roomId` | |
| `unarchive_group` | POST | `groups.unarchive` | `roomId` | |
| `delete_channel` | POST | `channels.delete` | `roomId` | |
| `delete_group` | POST | `groups.delete` | `roomId` | |
| `join_channel` | POST | `channels.join` | `roomId` or `channel` | |
| `leave_channel` | POST | `channels.leave` | `roomId` or `channel` | |
| `leave_group` | POST | `groups.leave` | `roomId` | |
| `invite_to_channel` | POST | `channels.invite` | `roomId`, `userId` | |
| `invite_to_group` | POST | `groups.invite` | `roomId`, `userId` | |
| `kick_from_channel` | POST | `channels.kick` | `roomId`, `userId` | |
| `kick_from_group` | POST | `groups.kick` | `roomId`, `userId` | |
| `set_channel_topic` | POST | `channels.setTopic` | `roomId`, `topic` | Requires `edit-room` |
| `set_group_topic` | POST | `groups.setTopic` | `roomId`, `topic` | Requires `edit-room` |
| `set_channel_description` | POST | `channels.setDescription` | `roomId`, `description` | |
| `set_group_description` | POST | `groups.setDescription` | `roomId`, `description` | |
| `get_channel_info` | GET | `channels.info` | `roomId` or `channel` | |
| `get_group_info` | GET | `groups.info` | `roomId` | |
| `get_channel_members` | GET | `channels.members` | `roomId` or `roomName`, `count`, `offset` | Broadcast channels need `view-broadcast-member-list` |
| `get_group_members` | GET | `groups.members` | `roomId`, `count`, `offset` | |
| `rename_channel` | POST | `channels.rename` | `roomId`, `name` | |
| `rename_group` | POST | `groups.rename` | `roomId`, `name` | |

#### Example: `GET /api/v1/channels.members`

Query: `?roomId=ROOM_ID&count=50`

Response:

```json
{
  "members": [
    { "_id": "...", "username": "alice", "status": "online", "name": "Alice" }
  ],
  "count": 50,
  "offset": 0,
  "total": 120,
  "success": true
}
```

#### Example: `POST /api/v1/channels.setTopic`

```json
{
  "roomId": "ROOM_ID",
  "topic": "Discuss all of the testing"
}
```

Response: `{ "topic": "...", "success": true }`.

### User / presence operations

| Candidate tool | Method | Endpoint | Body / Query | Notes |
| --- | --- | --- | --- | --- |
| `get_me` | GET | `me` | | Current authenticated user |
| `set_user_status` | POST | `users.setStatus` | `status` (`online`, `away`, `offline`, `busy`) | |
| `get_user_presence` | GET | `users.getPresence` | `userId` | |
| `set_user_avatar` | POST | `users.setAvatar` | `avatarUrl` or multipart `image` | |
| `update_user_info` | POST | `users.update` | `userId` + fields | Admin |
| `delete_user` | POST | `users.delete` | `userId` | Admin |
| `create_user` | POST | `users.create` | `email`, `name`, `password`, `username`, `roles` | Admin |
| `generate_pat` | POST | `users.generatePersonalAccessToken` | `tokenName` | |
| `list_pat` | GET | `users.getPersonalAccessTokens` | | |
| `remove_pat` | POST | `users.removePersonalAccessToken` | `tokenName` | |

### Subscription / read-state operations

| Candidate tool | Method | Endpoint | Body / Query | Notes |
| --- | --- | --- | --- | --- |
| `mark_channel_read` | POST | `subscriptions.read` | `rid` | |
| `mark_channel_unread` | POST | `subscriptions.unread` | `roomId` | |
| `get_all_subscriptions` | GET | `subscriptions.get` | | Already used for unread only |

---

## Full endpoint index

The complete Rocket.Chat REST API is organized below by functional area. Each
item links to the official endpoint page for method, parameter, and response
details.

### Authentication

- [Environment](https://developer.rocket.chat/apidocs/environment.md)
- [Step 1: Register Yourself as a User](https://developer.rocket.chat/apidocs/step-1-register-yourself-as-a-user.md)
- [Step 2: Authenticate Your User](https://developer.rocket.chat/apidocs/step-2-authenticate-your-user.md)
- [Step 3: Retrieve Your User Information](https://developer.rocket.chat/apidocs/step-3-retrieve-your-user-information.md)
- [Login with Username and Password](https://developer.rocket.chat/apidocs/login-with-username-and-password.md)
- [Login with Facebook](https://developer.rocket.chat/apidocs/login-with-facebook.md)
- [Login with Twitter](https://developer.rocket.chat/apidocs/login-with-twitter.md)
- [Login with Google](https://developer.rocket.chat/apidocs/login-with-google.md)
- [Logout](https://developer.rocket.chat/apidocs/logout.md)

### Users

- [Get Profile Information](https://developer.rocket.chat/apidocs/get-profile-information.md)
- [Enable 2FA via Email](https://developer.rocket.chat/apidocs/enable-2fa-via-email.md)
- [Send 2FA Email Code](https://developer.rocket.chat/apidocs/request-a-new-email-code.md)
- [Disable 2FA via Email](https://developer.rocket.chat/apidocs/disable-2fa-via-email.md)
- [Send Two-Factor Challenge Email Code](https://developer.rocket.chat/apidocs/send-two-factor-challenge-email-code.md)
- [Verify Two-Factor Challenge](https://developer.rocket.chat/apidocs/verify-two-factor-challenge.md)
- [Create User](https://developer.rocket.chat/apidocs/create-user.md)
- [Register User](https://developer.rocket.chat/apidocs/register-user.md)
- [Update User Details](https://developer.rocket.chat/apidocs/update-user.md)
- [Update Own Basic Information](https://developer.rocket.chat/apidocs/update-own-basic-information.md)
- [Get User's Info](https://developer.rocket.chat/apidocs/get-users-info.md)
- [Get Users List](https://developer.rocket.chat/apidocs/get-users-list.md)
- [List Users by Status](https://developer.rocket.chat/apidocs/list-users-by-status.md)
- [Set User Avatar](https://developer.rocket.chat/apidocs/set-user-avatar.md)
- [Get User Avatar](https://developer.rocket.chat/apidocs/get-user-avatar.md)
- [Reset Avatar](https://developer.rocket.chat/apidocs/reset-avatar.md)
- [Set User Status](https://developer.rocket.chat/apidocs/set-user-status.md)
- [Get Status](https://developer.rocket.chat/apidocs/get-status.md)
- [Set User's Status Active](https://developer.rocket.chat/apidocs/set-users-status-active.md)
- [Deactivate Idle Users](https://developer.rocket.chat/apidocs/deactivate-idle-users.md)
- [Get Specific User's Presence](https://developer.rocket.chat/apidocs/get-specific-users-presence.md)
- [Get Users Presence](https://developer.rocket.chat/apidocs/get-users-presence.md)
- [Delete User](https://developer.rocket.chat/apidocs/delete-user.md)
- [Delete Own Account](https://developer.rocket.chat/apidocs/delete-own-account.md)
- [Create Users Token](https://developer.rocket.chat/apidocs/create-users-token.md)
- [Get User's Preferences](https://developer.rocket.chat/apidocs/get-users-preferences.md)
- [Set User Preferences](https://developer.rocket.chat/apidocs/set-user-preferences.md)
- [Forgot Password](https://developer.rocket.chat/apidocs/forgot-password.md)
- [Get Username Suggestion](https://developer.rocket.chat/apidocs/get-username-suggestion.md)
- [Generate Personal Access Token](https://developer.rocket.chat/apidocs/generate-personal-access-token.md)
- [Regenerate Personal Access Token](https://developer.rocket.chat/apidocs/regenerate-personal-access-token.md)
- [Get Personal Access Tokens](https://developer.rocket.chat/apidocs/get-personal-access-tokens.md)
- [Remove Personal Access Token](https://developer.rocket.chat/apidocs/remove-personal-access-token.md)
- [Request Data Download](https://developer.rocket.chat/apidocs/request-data-download.md)
- [Logout Other Clients](https://developer.rocket.chat/apidocs/logout-other-clients.md)
- [Autocomplete User](https://developer.rocket.chat/apidocs/autocomplete-user.md)
- [Remove Other Tokens](https://developer.rocket.chat/apidocs/remove-other-tokens.md)
- [Reset Users E2E Key](https://developer.rocket.chat/apidocs/reset-users-e2e-key.md)
- [Reset Users TOTP](https://developer.rocket.chat/apidocs/reset-users-totp.md)
- [List User's Teams](https://developer.rocket.chat/apidocs/list-users-teams.md)
- [Report User](https://developer.rocket.chat/apidocs/report-user.md)
- [Logout User](https://developer.rocket.chat/apidocs/logout-user.md)
- [Get Avatars](https://developer.rocket.chat/apidocs/get-avatars.md)
- [Check Username Availability](https://developer.rocket.chat/apidocs/check-username-availability.md)
- [Send Welcome Email to User](https://developer.rocket.chat/apidocs/send-user-welcome-email.md)
- [Send Invitation Email](https://developer.rocket.chat/apidocs/send-invitation-email.md)
- [Send Email Verification](https://developer.rocket.chat/apidocs/send-email-verification.md)
- [Get Avatar Suggestion](https://developer.rocket.chat/apidocs/get-avatar-suggestion.md)
- [LDAP Sync](https://developer.rocket.chat/apidocs/ldap-sync.md)
- [Test LDAP Connection](https://developer.rocket.chat/apidocs/test-ldap-connection.md)
- [Test LDAP User Search](https://developer.rocket.chat/apidocs/test-ldap-user-search.md)

### Roles & Permissions

- [List All Permissions](https://developer.rocket.chat/apidocs/list-all-permissions.md)
- [Update Permissions](https://developer.rocket.chat/apidocs/update-permissions.md)
- [Create Role](https://developer.rocket.chat/apidocs/create-role.md)
- [Update Role](https://developer.rocket.chat/apidocs/update-role.md)
- [Assign Role to User](https://developer.rocket.chat/apidocs/assign-role-to-user.md)
- [Get Users of a Role](https://developer.rocket.chat/apidocs/get-users-of-a-role.md)
- [Get Roles](https://developer.rocket.chat/apidocs/get-roles.md)
- [Get Updated Roles](https://developer.rocket.chat/apidocs/get-updated-roles.md)
- [Delete Role](https://developer.rocket.chat/apidocs/delete-role.md)
- [Remove Role from User](https://developer.rocket.chat/apidocs/remove-role-from-user.md)
- [Get Users in Public Roles](https://developer.rocket.chat/apidocs/get-users-in-public-roles.md)

### Groups

- [Get Group Online Users](https://developer.rocket.chat/apidocs/get-group-online-users.md)
- [Get Group Integrations](https://developer.rocket.chat/apidocs/get-group-integrations.md)
- [Add All Users to Group](https://developer.rocket.chat/apidocs/add-all-users-to-group.md)
- [Add Group Leader](https://developer.rocket.chat/apidocs/add-group-leader.md)
- [Add Group Moderator](https://developer.rocket.chat/apidocs/add-group-moderator.md)
- [Add Group Owner](https://developer.rocket.chat/apidocs/add-group-owner.md)
- [Archive a Group](https://developer.rocket.chat/apidocs/archive-a-group.md)
- [Close Group](https://developer.rocket.chat/apidocs/close-group.md)
- [Get Group Counters](https://developer.rocket.chat/apidocs/get-group-counters.md)
- [Create Group](https://developer.rocket.chat/apidocs/create-group.md)
- [Delete Group](https://developer.rocket.chat/apidocs/delete-group.md)
- [Get Group History](https://developer.rocket.chat/apidocs/get-group-history.md)
- [Get Group Information](https://developer.rocket.chat/apidocs/get-group-information.md)
- [Invite Users to Group](https://developer.rocket.chat/apidocs/invite-users-to-group.md)
- [Remove User from Group](https://developer.rocket.chat/apidocs/remove-user-from-group.md)
- [Get List of User Groups](https://developer.rocket.chat/apidocs/get-list-of-user-groups.md)
- [Get Groups](https://developer.rocket.chat/apidocs/get-groups.md)
- [List Group Members](https://developer.rocket.chat/apidocs/list-group-members.md)
- [Get Group Messages](https://developer.rocket.chat/apidocs/get-group-messages.md)
- [Get Group Moderators](https://developer.rocket.chat/apidocs/get-group-moderators.md)
- [Add Group to List](https://developer.rocket.chat/apidocs/add-group-to-list.md)
- [Remove Group Leader](https://developer.rocket.chat/apidocs/remove-group-leader.md)
- [Remove Group Moderator](https://developer.rocket.chat/apidocs/remove-group-moderator.md)
- [Remove Group Owner](https://developer.rocket.chat/apidocs/remove-group-owner.md)
- [Rename Group](https://developer.rocket.chat/apidocs/rename-group.md)
- [Set Group Announcement](https://developer.rocket.chat/apidocs/set-group-announcement.md)
- [Set Group Custom Fields](https://developer.rocket.chat/apidocs/sets-group-custom-fields.md)
- [Set Group Description](https://developer.rocket.chat/apidocs/set-group-description.md)
- [Set Group Purpose](https://developer.rocket.chat/apidocs/set-group-purpose.md)
- [Set Group as Read Only](https://developer.rocket.chat/apidocs/set-group-as-read-only.md)
- [Set Group Topic](https://developer.rocket.chat/apidocs/set-group-topic.md)
- [Set Group Type](https://developer.rocket.chat/apidocs/set-group-type.md)
- [Unarchive Group](https://developer.rocket.chat/apidocs/unarchive-group.md)
- [Get Group Files](https://developer.rocket.chat/apidocs/get-group-files.md)
- [Set Group as Encrypted](https://developer.rocket.chat/apidocs/set-group-as-encrypted.md)
- [Convert a Group to Team](https://developer.rocket.chat/apidocs/convert-a-group-to-team.md)
- [List Group Roles](https://developer.rocket.chat/apidocs/list-group-roles.md)
- [Leave Group](https://developer.rocket.chat/apidocs/leave-group-1.md)

### Channels

- [Create Channel](https://developer.rocket.chat/apidocs/create-channel.md)
- [Add all Users to a Channel](https://developer.rocket.chat/apidocs/add-all-users-to-a-channel.md)
- [Add Channel Leader](https://developer.rocket.chat/apidocs/add-channel-leader.md)
- [Add Channel Moderator](https://developer.rocket.chat/apidocs/add-channel-moderator.md)
- [Add Channel Owner](https://developer.rocket.chat/apidocs/add-channel-owner.md)
- [Read Channel Messages Anonymously](https://developer.rocket.chat/apidocs/read-channel-messages-anonymously.md)
- [Archive Channel](https://developer.rocket.chat/apidocs/archive-channel.md)
- [Close Channel](https://developer.rocket.chat/apidocs/close-channel.md)
- [Get Channel Counters](https://developer.rocket.chat/apidocs/get-channel-counters.md)
- [Delete Channel](https://developer.rocket.chat/apidocs/delete-channel.md)
- [Get Channel Files](https://developer.rocket.chat/apidocs/get-channel-files.md)
- [Get Channel History](https://developer.rocket.chat/apidocs/get-channel-history.md)
- [Get Channel Information](https://developer.rocket.chat/apidocs/get-channel-information.md)
- [Add Users to Channel](https://developer.rocket.chat/apidocs/add-users-to-channel.md)
- [Join a Channel](https://developer.rocket.chat/apidocs/join-a-channel.md)
- [Remove User from Channel](https://developer.rocket.chat/apidocs/remove-user-from-channel.md)
- [Leave Channel](https://developer.rocket.chat/apidocs/leave-channel.md)
- [Get List of Joined Channels](https://developer.rocket.chat/apidocs/get-list-of-joined-channels.md)
- [Get Channel List](https://developer.rocket.chat/apidocs/get-channel-list.md)
- [Get Members of a Channel](https://developer.rocket.chat/apidocs/get-members-of-a-channel.md)
- [Get Channel Messages](https://developer.rocket.chat/apidocs/get-channel-messages.md)
- [Get Channel Moderators](https://developer.rocket.chat/apidocs/get-channel-moderators.md)
- [List Online Users in a Channel](https://developer.rocket.chat/apidocs/list-online-users-in-a-channel.md)
- [Add Channel to User List](https://developer.rocket.chat/apidocs/add-channel-to-user-list.md)
- [Remove Channel Leader](https://developer.rocket.chat/apidocs/remove-channel-leader.md)
- [Remove Channel Moderator](https://developer.rocket.chat/apidocs/remove-channel-moderator.md)
- [Remove Channel Owner](https://developer.rocket.chat/apidocs/remove-channel-owner.md)
- [Rename a Channel](https://developer.rocket.chat/apidocs/rename-a-channel.md)
- [Get Channel Roles](https://developer.rocket.chat/apidocs/get-channel-roles.md)
- [Set Channel Announcement](https://developer.rocket.chat/apidocs/set-channel-announcement.md)
- [Set Channel Custom Fields](https://developer.rocket.chat/apidocs/set-channel-custom-fields.md)
- [Set Default Channel](https://developer.rocket.chat/apidocs/set-default-channel.md)
- [Set Channel Description](https://developer.rocket.chat/apidocs/set-channel-description.md)
- [Set Channel Join Code](https://developer.rocket.chat/apidocs/set-channel-join-code.md)
- [Set Channel Purpose](https://developer.rocket.chat/apidocs/set-channel-purpose.md)
- [Set Channel ReadOnly](https://developer.rocket.chat/apidocs/set-channel-readonly.md)
- [Set Channel Topic](https://developer.rocket.chat/apidocs/set-channel-topic.md)
- [Set Channel Type](https://developer.rocket.chat/apidocs/set-channel-type.md)
- [Unarchive a Channel](https://developer.rocket.chat/apidocs/unarchive-a-channel.md)
- [Get User Mentions in a Channel](https://developer.rocket.chat/apidocs/get-all-user-mentions-in-a-channel.md)
- [Get Channel Integrations](https://developer.rocket.chat/apidocs/get-channel-integrations.md)
- [Convert Channel to Team](https://developer.rocket.chat/apidocs/convert-channel-to-team.md)

### Rooms

- [Set Room Notifications](https://developer.rocket.chat/apidocs/set-room-notifications.md)
- [Get All Room Admins](https://developer.rocket.chat/apidocs/get-all-room-admins.md)
- [Clear Room History](https://developer.rocket.chat/apidocs/clear-room-history.md)
- [Get Room Information](https://developer.rocket.chat/apidocs/get-room-information.md)
- [Get Room Discussions](https://developer.rocket.chat/apidocs/get-room-discussions.md)
- [Get Rooms](https://developer.rocket.chat/apidocs/get-rooms.md)
- [Leave Room](https://developer.rocket.chat/apidocs/leave-room.md)
- [Delete Room](https://developer.rocket.chat/apidocs/delete-room.md)
- [Favorite/Unfavourite a Room](https://developer.rocket.chat/apidocs/favoriteunfavourite-a-room.md)
- [Autocomplete Room Name for Team](https://developer.rocket.chat/apidocs/autocomplete-room-name-for-team.md)
- [Autocomplete Room Name for Private and Public Rooms](https://developer.rocket.chat/apidocs/autocomplete-private-channel.md)
- [Get Admin of Room](https://developer.rocket.chat/apidocs/get-admin-of-room.md)
- [Save Room Settings](https://developer.rocket.chat/apidocs/save-room-settings.md)
- [Change Room Archive State](https://developer.rocket.chat/apidocs/change-room-archive-state.md)
- [Export Room](https://developer.rocket.chat/apidocs/export-room.md)
- [Create Discussion](https://developer.rocket.chat/apidocs/create-discussion.md)
- [Check if Room Name Exists](https://developer.rocket.chat/apidocs/check-if-room-name-exists.md)
- [Mute User in Room](https://developer.rocket.chat/apidocs/mute-user-in-room.md)
- [Unmute User in Room](https://developer.rocket.chat/apidocs/unmute-user-in-room.md)
- [Admin Autocomplete Room Name for Private and Public Rooms](https://developer.rocket.chat/apidocs/admin-autocomplete-room-name-for-private-and-public-rooms.md)
- [Get Room Images](https://developer.rocket.chat/apidocs/get-room-images.md)
- [Audit and Get Room Members](https://developer.rocket.chat/apidocs/audit-rooms.md)
- [Upload Media Files to a Room](https://developer.rocket.chat/apidocs/upload-media-files-to-a-room.md)
- [Get Room Members Ordered by Role](https://developer.rocket.chat/apidocs/get-room-members-ordered-by-role.md)
- [Hide Room](https://developer.rocket.chat/apidocs/hide-room.md)
- [Get Room Roles](https://developer.rocket.chat/apidocs/get-room-roles.md)
- [Check Room Member](https://developer.rocket.chat/apidocs/check-room-member.md)
- [Autocomplete Room Name With Pagination](https://developer.rocket.chat/apidocs/autocomplete-room-name-with-pagination.md)
- [Confirm Uploaded File](https://developer.rocket.chat/apidocs/check-uploaded-file.md)
- [Delete Uploaded File](https://developer.rocket.chat/apidocs/delete-uploaded-file.md)

### Subscriptions

- [Get All Subscriptions](https://developer.rocket.chat/apidocs/get-all-subscriptions.md)
- [Get Subscription Room](https://developer.rocket.chat/apidocs/get-subscription-room.md)
- [Mark Channel as Read](https://developer.rocket.chat/apidocs/mark-channel-as-read.md)
- [Mark Channel as Unread](https://developer.rocket.chat/apidocs/mark-channel-as-unread.md)

### Teams

- [Create a New Team](https://developer.rocket.chat/apidocs/create-a-new-team.md)
- [Get List of All Teams](https://developer.rocket.chat/apidocs/get-list-of-all-teams.md)
- [Get List of Teams](https://developer.rocket.chat/apidocs/get-list-of-teams.md)
- [Get Team Info](https://developer.rocket.chat/apidocs/get-team-info.md)
- [Update a Team](https://developer.rocket.chat/apidocs/update-a-team.md)
- [Add Members to the Team](https://developer.rocket.chat/apidocs/add-members-to-the-team.md)
- [List Team Members](https://developer.rocket.chat/apidocs/list-team-members.md)
- [Update Team Member Info](https://developer.rocket.chat/apidocs/update-team-member-info.md)
- [Leave a Team](https://developer.rocket.chat/apidocs/leave-a-team.md)
- [Remove Member from Team](https://developer.rocket.chat/apidocs/remove-member-from-team.md)
- [Delete a Team](https://developer.rocket.chat/apidocs/delete-a-team.md)
- [Autocomplete Team](https://developer.rocket.chat/apidocs/autocomplete-team.md)
- [Convert Team to Channel](https://developer.rocket.chat/apidocs/convert-team-to-channel.md)
- [Add Rooms to a Team](https://developer.rocket.chat/apidocs/add-rooms-to-a-team.md)
- [Remove Room from the Team](https://developer.rocket.chat/apidocs/remove-room-from-the-team.md)
- [Update Room in a Team](https://developer.rocket.chat/apidocs/update-room-in-a-team.md)
- [List Rooms of a Team](https://developer.rocket.chat/apidocs/list-rooms-of-a-team.md)
- [List User Rooms of a Team](https://developer.rocket.chat/apidocs/list-user-rooms-of-a-team.md)
- [List Rooms and Discussions of a Team](https://developer.rocket.chat/apidocs/list-rooms-and-discussions-of-a-team.md)

### Messages

- [Delete Chat Message](https://developer.rocket.chat/apidocs/delete-chat-message.md)
- [React to Message](https://developer.rocket.chat/apidocs/react-to-message.md)
- [Update Message](https://developer.rocket.chat/apidocs/update-message.md)
- [Report Message](https://developer.rocket.chat/apidocs/report-message.md)
- [Follow Message](https://developer.rocket.chat/apidocs/follow-message.md)
- [Unfollow Message](https://developer.rocket.chat/apidocs/unfollow-message.md)
- [Get Message](https://developer.rocket.chat/apidocs/get-message.md)
- [Get Deleted Messages](https://developer.rocket.chat/apidocs/get-deleted-messages.md)
- [Get Discussions of A Room](https://developer.rocket.chat/apidocs/get-discussions-of-a-room.md)
- [Get Mentioned Messages](https://developer.rocket.chat/apidocs/get-mentioned-messages.md)
- [Get Message Read Receipts](https://developer.rocket.chat/apidocs/get-message-read-receipts.md)
- [Get Pinned Messages](https://developer.rocket.chat/apidocs/get-pinned-messages.md)
- [Get Starred Messages](https://developer.rocket.chat/apidocs/get-starred-messages.md)
- [Get Thread Messages](https://developer.rocket.chat/apidocs/get-thread-messages.md)
- [Ignore User](https://developer.rocket.chat/apidocs/ignore-user.md)
- [Pin Message](https://developer.rocket.chat/apidocs/pin-message.md)
- [Unpin a message](https://developer.rocket.chat/apidocs/unpin-a-message.md)
- [Post Message](https://developer.rocket.chat/apidocs/post-message.md)
- [Send Message](https://developer.rocket.chat/apidocs/send-message.md)
- [Star Message](https://developer.rocket.chat/apidocs/star-message.md)
- [Unstar Message](https://developer.rocket.chat/apidocs/unstar-message.md)
- [Sync Thread List](https://developer.rocket.chat/apidocs/sync-thread-list.md)
- [Sync Thread Messages](https://developer.rocket.chat/apidocs/sync-thread-messages.md)
- [Sync Messages](https://developer.rocket.chat/apidocs/sync-messages.md)
- [List Threads](https://developer.rocket.chat/apidocs/list-threads.md)
- [Get URL Preview](https://developer.rocket.chat/apidocs/get-url-preview.md)
- [Search Message](https://developer.rocket.chat/apidocs/search-message.md)

### Direct Messages

- [Close DM](https://developer.rocket.chat/apidocs/close-dm-1.md)
- [Get DM Counters](https://developer.rocket.chat/apidocs/get-dm-counters-1.md)
- [Create DM](https://developer.rocket.chat/apidocs/create-dm-1.md)
- [Delete DM](https://developer.rocket.chat/apidocs/delete-dm-1.md)
- [Get DM Files](https://developer.rocket.chat/apidocs/get-dm-files.md)
- [DM History](https://developer.rocket.chat/apidocs/dm-history-1.md)
- [List All DMs](https://developer.rocket.chat/apidocs/list-all-dms.md)
- [List DMs](https://developer.rocket.chat/apidocs/list-dms-1.md)
- [List DM Members](https://developer.rocket.chat/apidocs/list-dm-members-1.md)
- [List DM Messages](https://developer.rocket.chat/apidocs/list-dm-messages.md)
- [Message Others](https://developer.rocket.chat/apidocs/message-others.md)
- [Open DM](https://developer.rocket.chat/apidocs/open-dm-1.md)
- [Set DM Topic](https://developer.rocket.chat/apidocs/set-dm-topic-1.md)

### Integrations, OAuth, WebDAV

- [Create Integration](https://developer.rocket.chat/apidocs/create-integration.md)
- [Get Integration](https://developer.rocket.chat/apidocs/get-integration.md)
- [Get Integration History](https://developer.rocket.chat/apidocs/get-integration-history.md)
- [Get List of Integrations](https://developer.rocket.chat/apidocs/get-list-of-integrations.md)
- [Update Integration](https://developer.rocket.chat/apidocs/update-integration.md)
- [Remove Integration](https://developer.rocket.chat/apidocs/remove-integration-1.md)
- [Get WebDAV Accounts](https://developer.rocket.chat/apidocs/get-webdav-accounts.md)
- [Remove WebDAV Account](https://developer.rocket.chat/apidocs/remove-webdav-account.md)
- [Create OAuth App](https://developer.rocket.chat/apidocs/create-oauth-app.md)
- [Update OAuth App](https://developer.rocket.chat/apidocs/update-oauth-app.md)
- [Get List of OAuth Apps](https://developer.rocket.chat/apidocs/get-list-of-oauth-apps.md)
- [Get OAuth App](https://developer.rocket.chat/apidocs/get-oauth-app.md)
- [Delete OAuth App](https://developer.rocket.chat/apidocs/delete-oauth-app.md)

### Omnichannel / Livechat

See the official docs for the complete Omnichannel section. Key areas:

- Agents & Managers
- Visitors & Contacts
- Rooms
- Departments
- Analytics & Reporting
- Inquiries
- Custom Fields & Business Hours
- Priorities, Tags, Units & SLA
- Canned Responses
- Transcripts
- Webhooks & Configurations
- Livechat Messages

Official category root: https://developer.rocket.chat/apidocs

### Settings & Server Administration

- [Get Public Settings](https://developer.rocket.chat/apidocs/get-public-settings.md)
- [Get OAuth Settings](https://developer.rocket.chat/apidocs/get-oauth-settings.md)
- [Get Private Settings](https://developer.rocket.chat/apidocs/get-private-settings.md)
- [Add Custom OAuth](https://developer.rocket.chat/apidocs/add-custom-oauth.md)
- [Update Setting](https://developer.rocket.chat/apidocs/update-setting.md)
- [Get Setting](https://developer.rocket.chat/apidocs/get-setting.md)
- [Get OAuth Service Configuration](https://developer.rocket.chat/apidocs/get-oauth-service-configuration.md)
- [Get Statistics List](https://developer.rocket.chat/apidocs/get-statistics-list.md)
- [Get Last Statistics](https://developer.rocket.chat/apidocs/get-last-statistics.md)

### Realtime APIs

Rocket.Chat also exposes WebSocket-based realtime APIs. These are out of scope
for the current stdio MCP server but are listed in the official index under
[Realtime APIs](https://developer.rocket.chat/apidocs).

---

## Notes

- Rate limiting is enabled by default. Watch for response headers
  `x-ratelimit-limit`, `x-ratelimit-remaining`, and `x-ratelimit-reset`.
- Some endpoints always trigger rate limiting and cannot be disabled.
- Most endpoints require the caller to have the matching `view-*`, `edit-*`,
  or `manage-*` permission. The official page for each endpoint lists the exact
  permission.
- When expanding tools, prefer endpoints that accept `roomId` or `roomName`
  consistently to minimize the room-resolution logic already in
  `RocketChatAPI.resolve_room_id`.
