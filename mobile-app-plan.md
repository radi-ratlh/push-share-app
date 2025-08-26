# React Native Mobile App Requirements (Expo)

## Introduction

This mobile app is the companion frontend for the push-share system, designed to let users quickly send and receive text and files across their registered devices. It focuses on fast handoff (phone ↔ desktop/laptop/tablet), minimal friction (no accounts; device identity only), reliable delivery via push notifications, and a clean, chat-like experience that works offline and recovers gracefully when the network returns.

## 1) Goals and Scope

- Goal: Mobile-first client for your device-to-device messaging system.
- Platforms: iOS 15+ and Android 8+ (Expo SDK latest LTS).
- Identity: device-based (no accounts). Each install registers a device and an Expo push token.
- Core features:
  - Onboarding and device registration
  - Device list + “All devices” merged feed
  - Conversations (1:1)
  - Send text and files
  - Delivery ticks (single pending, double green delivered)
  - Date-grouped message list + “New” divider
  - Delete (soft by default, optional permanent)
  - Offline queue and retry
  - Push notifications (Expo) for message delivery; no WebSockets

## 2) Architecture Overview

- App (Expo RN):
  - Uses expo-notifications to get permissions and Expo push token.
  - Registers device + token with backend.
  - Receives notifications:
    - Foreground: in-app banner + insert message into feed.
    - Background/terminated: tapping navigates to the conversation; silent/data payload triggers background refresh where supported.
- Backend:
  - Persists Expo push token per device.
  - On new Message rows for a recipient device, sends push via Expo Push API.
  - Provides REST endpoints for listing, sending, acknowledging delivered, deleting, and fetching attachments.

## Tech stack and libraries (latest stable)

- Runtime and tooling
  - Expo (SDK latest LTS) with React Native (latest stable)
  - React Navigation (latest) for screen/navigation stacks
  - @tanstack/react-query v5 (latest) for server-state caching and retries
  - expo-notifications (latest) for permissions and Expo push token handling
  - expo-document-picker / expo-image-picker (latest) for file selection
  - expo-file-system (latest) for downloads/caching
  - react-native-permissions (latest) for permission flows
- UI layer
  - React Native Paper (latest, MD3) as the component library: Appbar, List, Button, Chip, Snackbar, Dialog, Avatar, Card, ActivityIndicator (https://reactnativepaper.com)
  - NativeWind (latest) for Tailwind-like utility classes in RN (https://www.nativewind.dev)
    - Configure Tailwind config and the NativeWind Babel plugin
    - Use utility classes for spacing, layout, color, and theming alongside Paper
  - react-native-vector-icons / @expo/vector-icons (latest) for iconography (Paper compatible)
- State/persistence
  - AsyncStorage or MMKV (latest) for lightweight local storage (deviceId, outbox)
  - Secure storage (expo-secure-store latest) for deviceId/token where appropriate

## 3) Data Model (server-side reference)

- message: id, sender_device_id, recipient_device_id, kind(TEXT|FILE), text_payload, created_at (ISO), status(PENDING|DELIVERED|READ optional), deleted(boolean).
- message_attachment: id, message_id, filename, mime, size_bytes, file_path, deleted(boolean).
- device: id, name, platform, expo_push_token (nullable), last_seen_at.

## 4) Push Notifications (Expo) Flow

- Token lifecycle:
  - App requests permission on first launch; retrieves token via expo-notifications.
  - POST/PUT token to backend; DELETE token on logout/uninstall (best-effort).
  - Backend stores per-device expo_push_token.
- When a message is created for a device:
  - Backend sends an Expo message to that device’s token.
- Notification styles:
  - Foreground: in-app banner + insert/animate message in the list.
  - Background/terminated: tapping navigates to conversation or the “All devices” feed.
- Payloads sent by backend to Expo:

```json
{
  "to": "ExponentPushToken[xxxxxxxxxxxxxxxxxxxxxx]",
  "title": "My Phone",
  "body": "New file: project_brief.pdf",
  "sound": "default",
  "priority": "high",
  "data": {
    "type": "message",
    "messageId": "bb0e8c55-3a5e-4c78-9e67-3a9b3a65f2c1",
    "senderDeviceId": "a1",
    "recipientDeviceId": "b1",
    "kind": "FILE",
    "attachmentId": "8b7a0b96-1d68-4f8c-86d6-2a0c3d61f6f0",
    "snippet": "project_brief.pdf",
    "createdAt": "2025-08-20T10:10:05.123Z"
  }
}
```

- App handling:
  - If app is foreground: show lightweight in-app banner and call delivered ACK.
  - If app is background/terminated: route via deep link using data fields; after navigating, ACK delivery.

## 5) API Specification (HTTP, JSON)

### Common

- Base URL: set via ENV, e.g., https://api.example.com
- Headers: Content-Type: application/json unless multipart upload
- Time: All timestamps are ISO-8601 UTC strings
- Errors: 4xx/5xx JSON

```json
{
  "error": "BadRequest",
  "message": "Missing senderDeviceId"
}
```

### Devices

- POST /devices/register

```json
// request
{
  "name": "iPhone 15",
  "platform": "iOS",
  "expoPushToken": "ExponentPushToken[xxxxxxxx]",
  "clientVersion": "1.0.0"
}
// response 201
{
  "id": "d-123",
  "name": "iPhone 15",
  "platform": "iOS",
  "expoPushToken": "ExponentPushToken[xxxxxxxx]",
  "createdAt": "2025-08-20T09:32:11Z"
}
```

- PUT /devices/{deviceId}

```json
// request (any subset to update)
{
  "name": "My Phone",
  "expoPushToken": "ExponentPushToken[yyyyyyyy]"
}
// response 200
{ "ok": true }
```

- DELETE /devices/{deviceId}/push-token

```json
// response 204 (token cleared server-side)
```

- GET /devices

```json
// response 200
[
  {
    "id": "d-123",
    "name": "My Phone",
    "platform": "iOS",
    "lastSeenAt": "2025-08-20T10:20:00Z"
  },
  {
    "id": "d-456",
    "name": "Work Laptop",
    "platform": "macOS",
    "lastSeenAt": "2025-08-20T09:55:00Z"
  }
]
```

### Messages (text)

- POST /items (text)

```json
// request
{
  "senderDeviceId": "d-123",
  "recipients": ["d-456"],
  "kind": "TEXT",
  "text": "Got it, thanks!"
}
// response 201
{
  "messageIds": ["m-789"],         // one per recipient
  "status": "PENDING",
  "createdAt": "2025-08-20T10:12:00Z"
}
```

### Messages (file)

- POST /items (multipart)

Form fields

- senderDeviceId: string
- recipients: comma-separated device IDs, e.g., d-456 or d-456,d-789
- file: binary

```http
POST /items
Content-Type: multipart/form-data; boundary=----abcd
------abcd
Content-Disposition: form-data; name="senderDeviceId"

d-123
------abcd
Content-Disposition: form-data; name="recipients"

d-456
------abcd
Content-Disposition: form-data; name="file"; filename="project_brief.pdf"
Content-Type: application/pdf

<binary>
------abcd--
// response 201
{
  "messageIds": ["m-790"],
  "attachmentIds": ["a-555"],
  "filename": "project_brief.pdf",
  "sizeBytes": 345123
}
```

### Conversation and Feeds

- GET /items/conversation?deviceA=d-123&deviceB=d-456&limit=50&cursor=2025-08-19T23:59:59Z

```json
// response 200
{
  "items": [
    {
      "id": "m-700",
      "senderDeviceId": "d-456",
      "recipientDeviceId": "d-123",
      "kind": "TEXT",
      "text": "Meeting in 15 mins!",
      "createdAt": "2025-08-20T10:00:00Z",
      "status": "DELIVERED",
      "attachment": null
    },
    {
      "id": "m-701",
      "senderDeviceId": "d-123",
      "recipientDeviceId": "d-456",
      "kind": "FILE",
      "text": null,
      "createdAt": "2025-08-20T10:10:05Z",
      "status": "PENDING",
      "attachment": {
        "id": "a-900",
        "filename": "project_brief.pdf",
        "mime": "application/pdf",
        "sizeBytes": 345123
      }
    }
  ],
  "nextCursor": "2025-08-20T09:00:00Z"
}
```

- GET /items/conversation/group?deviceId=d-123&limit=50&cursor=…

Returns merged feed across all registered devices belonging to the system (same shape as above).

### Delivery and Read

- POST /items/delivered

```json
// request
{
  "deviceId": "d-123",
  "messageIds": ["m-700", "m-701"]
}
// response 200
{ "updated": 2 }
```

### Downloads

- GET /attachments/{attachmentId}

Binary stream (404 if soft-deleted)

### Deletes

- DELETE /items/{messageId}?permanent=false
- DELETE /attachments/{attachmentId}?permanent=false

```json
// response 204 on success
```

### App Config

- GET /app/config

```json
// response 200
{ "permanentDeleteEnabled": false, "maxUploadBytes": 26214400 }
```

## 6) UI/UX Requirements

### Global

- Theme: light/dark; accessible contrast; system-following.
- Haptics: light impact on send, soft on error, selection on long-press menu.
- Date grouping:
  - Divider chip before the first message of each day.
  - Labels: Today, Yesterday, or localized “MMM d, yyyy”.
- Delivery ticks:
  - My messages: bottom-right meta row shows hh:mm and:
    - Pending: single check (gray 500)
    - Delivered: double check (green 500)
- “New” divider: when opening a chat, insert a “New” chip above the first message newer than lastOpenAt.

- Component and styling approach
  - Use React Native Paper for structure and components: Appbar.Header, List.Item, FAB, Button, Dialog, Chip, Snackbar, ActivityIndicator, Avatar.
  - Use NativeWind for utility classes: padding, margin, flex, bg/fg colors, rounded corners, alignment. Example:
    - `<View className="flex-1 bg-background px-3 pt-2">`
    - `<View className="self-end max-w-[80%] rounded-2xl rounded-br-sm bg-primary/15 p-2">`
  - Integrate Paper theme with Tailwind colors: wire Paper’s MD3 theme to Tailwind config for consistent tokens (primary, secondary, background, surface, error).
  - Typography: Paper’s MD3 text styles; scale with system settings; ensure minimum touch targets (44x44).

### Screens

- Splash/Bootstrap

  - Reads stored deviceId and routes to Onboarding or Home.
  - If deviceId exists but backend has no such device, clear and re-onboard.

- Onboarding

  - Steps: name → register device (POST /devices/register with token).
  - Permission flow:
    - Request notification permission with rationale.
    - If denied: app still works, but show non-blocking banner to enable later.
  - Success: navigate to Home.

- Home

  - Drawer/List of devices:
    - “All devices” pinned first; tapping opens merged feed.
    - Each device: avatar initials, name, presence/last seen.
  - Tapping a device opens Conversation.
  - Pull-to-refresh refreshes device list.

- Conversation

  - Header:
    - Avatar + name for peer, or “All devices” title.
    - Overflow menu: Mute notifications (local), View device info.
  - Message list (only area that scrolls):
    - Bubbles:
  - Mine (self): Paper surfaceVariant color or `bg-primary/15` with NativeWind; `rounded-2xl rounded-br-sm`.
  - Theirs: `bg-muted`/surface color; `rounded-2xl rounded-bl-sm`.
    - Attachments:
      - File bubble shows filename, size, and a file icon; tap to open via OS viewer.
      - Show small download spinner while fetching.
    - Animation: new items slide/grow in.
    - Long-press bubble:
      - Actions: Copy text, Delete, Delete permanently (if enabled via /app/config), Share (OS sheet).
      - Soft delete is instant with Undo snackbar (5s). Permanent shows a confirm dialog.
    - Swipe left on my messages for quick delete (optional enhancement).
  - Composer (hidden in “All devices” feed):
    - Multiline input (auto-grow up to 6 lines); placeholder “Type a message”.
  - Enter/Return sends; Shift+Enter inserts newline (Android/iOS keyboard return behavior configured).
    - Attach button opens file picker; show selected file chip and upload progress.
    - Disabled when no peer selected or while sending/uploading.
    - Optimistic send:
      - Insert a local “sending” bubble immediately with a grey single-check.
      - Replace it with the server message on success; on failure, show “Tap to retry” state.

- Settings
  - API base URL and environment.
  - Theme mode.
  - Notification settings: enable/disable local banners; re-request permission.
  - About/version.

### Empty/Error states

- No devices: onboarding CTA.
- No messages: friendly illustration + “Start a conversation.”
- Errors: toast/banner with retry.

### Accessibility

- VoiceOver/TalkBack labels for buttons and message metadata (“Delivered at 10:12 PM”).
- Dynamic type support; minimum 44x44 touch targets.

## 7) Offline & Retry

- Outbox:
  - Persist pending sends (text and file metadata) locally.
  - Exponential backoff retry; resume on connectivity.
- File uploads:
  - Show progress; allow cancel; retry on failure.
- When offline:
  - Composer stays enabled; items queued with “Pending” badge.
  - Incoming: rely on pull-to-refresh when app regains network; background refresh if a data-only notification arrives.

## 8) Security & Privacy

- HTTPS in production.
- Store deviceId and token in secure storage (SecureStore/Keychain/Keystore).
- Do not keep downloaded files permanently unless user saves.
- No PII in logs; redact tokens.

## 9) Configuration

- App env:
  - EXPO_PUBLIC_API_BASE
  - EXPO_PUBLIC_WS_BASE (unused in MVP)
- Feature flags (from GET /app/config):
  - permanentDeleteEnabled: boolean
  - maxUploadBytes: number
- Platform permissions:
  - Notifications; Files/Photos as needed for attach/open.

## 10) API Usage Examples in App

Register device and token

```json
POST /devices/register
{
  "name": "Pixel 8",
  "platform": "Android",
  "expoPushToken": "ExponentPushToken[zzzzzzzz]",
  "clientVersion": "1.0.0"
}
```

Send text and optimistically render

```json
POST /items
{
  "senderDeviceId": "d-123",
  "recipients": ["d-456"],
  "kind": "TEXT",
  "text": "On my way."
}
```

Upload a file

```http
POST /items (multipart: senderDeviceId=d-123, recipients=d-456, file=@/path/file.pdf)
```

Mark delivered (on open or on notification receipt)

```json
POST /items/delivered
{ "deviceId": "d-456", "messageIds": ["m-701","m-702"] }
```

Soft delete

```http
DELETE /items/m-701
// 204
```

Permanent delete (if allowed)

```http
DELETE /items/m-701?permanent=true
// 204 or 403 if disabled
```

## 11) Acceptance Criteria

- Device registration persists deviceId + token; token refresh updates server.
- Receiving a notification while foreground inserts/animates message with single check; upon entering chat, delivered tick updates to double green after ACK call.
- Date chips display correctly (Today/Yesterday/MMM d, yyyy).
- Composer grows up to 6 lines; Enter sends; Shift+Enter newline.
- Attachments upload and appear with progress, then finalize.
- Soft delete hides message with Undo; permanent delete gated by config.
- “All devices” feed shows combined timeline; composer hidden there.
- Only the message list scrolls; header and composer remain fixed.

## 12) Next Steps

- Backend additions for push:
  - Persist expo_push_token per device.
  - Send Expo push on new message events.
  - Implement POST /items/delivered (batch).
- Mobile bootstrap:
  - Expo app init with latest Expo SDK; add latest: react-navigation, @tanstack/react-query, expo-notifications, expo-document-picker, expo-file-system, react-native-permissions.
  - Add React Native Paper (latest) and configure MD3 theme.
  - Add NativeWind (latest): install, Tailwind config, Babel plugin, theme tokens aligned with Paper.
  - Implement screens/components per spec and wire APIs.
- QA with a staging backend; validate notification routing and deep links.
