# Karas Messenger: Mobile API и Realtime Contracts

Документ фиксирует минимальный стабильный серверный контракт, достаточный для native iOS/Android клиентов без зависимости от web UI.

## 1. Базовые правила

- REST base URL: `<server-origin>/api`
- Auth transport: `Authorization: Bearer <jwt>`
- Socket transport: `socket.io`
- Предпочтительная socket auth-модель для mobile:
  - подключать socket с `auth.token = <jwt>`
  - после `connect` можно не дублировать identity через payload `from/userId`, сервер канонизирует user id из токена
- Backward-compatible fallback для старых клиентов:
  - после `connect` отправить `register(<userId>)`

## 2. Auth flow

### Регистрация

`POST /api/auth/register`

Request:

```json
{
  "username": "alice",
  "displayName": "Alice",
  "password": "secret",
  "email": "alice@example.com",
  "publicKey": {},
  "encryptedPrivateKey": "iv:ciphertext",
  "keySalt": "base64-salt"
}
```

Response:

```json
{
  "token": "jwt",
  "user": {
    "id": "uuid",
    "username": "alice",
    "name": "Alice",
    "displayName": "Alice",
    "avatar": "🙂",
    "bio": "",
    "phone": "",
    "email": "alice@example.com",
    "quickReactionEmoji": "❤️",
    "hasPassword": true,
    "hasKeyBackup": true,
    "authProvider": "local",
    "keyBackupType": "account_password",
    "needsE2eeBackupSetup": false,
    "hasE2eePublicKey": true
  },
  "encryptedPrivateKey": "iv:ciphertext",
  "keySalt": "base64-salt",
  "hasKeyBackup": true,
  "hasPassword": true,
  "authProvider": "local",
  "keyBackupType": "account_password",
  "needsE2eeBackupSetup": false,
  "hasKeys": true
}
```

### Логин

- `POST /api/auth/login`
- `POST /api/auth/google`

Оба endpoint возвращают тот же `AuthResponse`, что и регистрация.

### E2EE backup

- `GET /api/auth/e2ee-backup`
- `PUT /api/auth/e2ee-backup`
- `POST /api/auth/set-password`
- `POST /api/auth/change-password`

Для mobile это критично, если клиент хранит локальный private key и должен восстанавливать его после нового логина.

## 3. Основные DTO

### User DTO

```json
{
  "id": "uuid",
  "username": "alice",
  "name": "Alice",
  "displayName": "Alice",
  "avatar": "🙂",
  "bio": "",
  "phone": "",
  "email": "alice@example.com",
  "quickReactionEmoji": "❤️",
  "hasPassword": true,
  "hasKeyBackup": true,
  "authProvider": "local",
  "keyBackupType": "account_password",
  "needsE2eeBackupSetup": false,
  "hasE2eePublicKey": true
}
```

### Chat Summary DTO

Используется в `GET /api/chats`, `POST /api/chats`, `POST /api/groups`, `POST /api/channels`, `POST /api/channels/:id/subscribe`, `PUT /api/groups/:id`.

```json
{
  "id": "chat-id",
  "isGroup": true,
  "isChannel": false,
  "chatType": "group",
  "name": "Team",
  "avatar": "👥",
  "otherMembers": [
    {
      "id": "user-id",
      "username": "bob",
      "displayName": "Bob",
      "avatar": "🙂",
      "bio": "",
      "phone": "",
      "lastSeenAt": "2026-04-03T12:00:00.000Z"
    }
  ],
  "otherUserIds": ["user-id"],
  "lastMessage": {
    "text": "cipher-or-plain-text",
    "time": "2026-04-03T12:00:00.000Z",
    "senderId": "user-id",
    "iv": "base64-iv",
    "mediaUrl": "/uploads/file.jpg",
    "mediaType": "image"
  },
  "unreadCount": 3,
  "membersCount": 7,
  "allowSelfSubscribe": false,
  "createdAt": "2026-04-03T11:00:00.000Z",
  "lastSeenAt": "2026-04-03T12:00:00.000Z",
  "username": "bob",
  "bio": "",
  "phone": "",
  "isBlockedByMe": false,
  "hasBlockedMe": false,
  "isAdmin": true,
  "isOwner": false,
  "isPinned": false,
  "notificationsEnabled": true
}
```

### Message DTO

Используется в:

- `GET /api/chats/:id/messages`
- `GET /api/channels/:id/preview/messages`
- `POST /api/chats/:id/messages`
- socket `message:new:{userId}`

```json
{
  "id": "message-id",
  "senderId": "user-id",
  "clientRequestId": "client-uuid",
  "replyToMessageId": "message-id",
  "contentType": "text",
  "text": "ciphertext-or-plaintext",
  "iv": "base64-iv",
  "mediaUrl": "/uploads/file.jpg",
  "mediaType": "image",
  "mediaName": "file.jpg",
  "sticker": {
    "stickerId": "sticker-id",
    "packId": "pack-id",
    "format": "animated",
    "width": 256,
    "height": 256,
    "previewUrl": "/stickers-assets/preview.webp",
    "assetUrl": "/stickers-assets/file.tgs",
    "emoji": "🔥",
    "tags": ["fire"]
  },
  "createdAt": "2026-04-03T12:00:00.000Z",
  "read": false,
  "readAt": null,
  "isEdited": false,
  "senderName": "Alice",
  "senderAvatar": "🙂",
  "isMine": false,
  "reactions": [
    {
      "emoji": "❤️",
      "count": 2,
      "userIds": ["u1", "u2"]
    }
  ]
}
```

### Upload response

`POST /api/upload` multipart form-data c полем `file`

```json
{
  "mediaUrl": "/uploads/uuid.jpg",
  "mediaType": "image",
  "originalName": "photo.jpg"
}
```

`mediaUrl` сейчас сервер возвращает как относительный путь. Mobile-клиент должен резолвить его относительно origin сервера.

## 4. Mobile-критичные REST endpoints

### Инициализация сессии

- `GET /api/mobile/bootstrap`
- `GET /api/users/me`
- `GET /api/chats`
- `GET /api/chats/:id/messages`
- `GET /api/users/:id/public-key`
- `GET /api/auth/e2ee-backup`

Предпочтительный mobile bootstrap:

```json
{
  "contractsVersion": "mobile-v1",
  "serverTime": "2026-04-03T12:00:00.000Z",
  "user": { "...": "ClientUser DTO" },
  "e2ee": {
    "encryptedPrivateKey": "iv:ciphertext",
    "keySalt": "base64-salt",
    "hasKeyBackup": true,
    "keyBackupType": "account_password",
    "hasKeys": true
  },
  "chats": [{ "...": "Chat Summary DTO" }],
  "realtime": {
    "socketAuth": "bearer_preferred",
    "legacyRegisterEvent": "register",
    "pendingCallEvent": "call:pending:get"
  }
}
```

### Отправка и изменение контента

- `POST /api/chats/:id/messages`
- `PUT /api/messages/:id`
- `DELETE /api/messages/:id`
- `POST /api/upload`
- `POST /api/messages/:id/reactions`

Request для отправки сообщения:

```json
{
  "text": "ciphertext-or-plaintext",
  "iv": "base64-iv",
  "mediaUrl": "/uploads/uuid.jpg",
  "mediaType": "image",
  "mediaName": "photo.jpg",
  "replyToMessageId": "message-id",
  "clientRequestId": "client-uuid",
  "contentType": "text",
  "sticker": {
    "stickerId": "sticker-id",
    "packId": "pack-id"
  }
}
```

### Управление чатами и membership

- `POST /api/chats`
- `POST /api/groups`
- `POST /api/channels`
- `PUT /api/groups/:id`
- `GET /api/groups/:id/members`
- `POST /api/groups/:id/members`
- `DELETE /api/groups/:id/members/:userId`
- `PUT /api/groups/:id/members/:userId/admin`
- `DELETE /api/chats/:id`
- `PUT /api/chats/:id/preferences`

### Каналы

- `GET /api/channels/:id/preview`
- `GET /api/channels/:id/preview/messages`
- `POST /api/channels/:id/subscribe`
- `GET /api/channels/:id/stats`

### Push

- `GET /api/push/vapid-public-key`
- `POST /api/push/devices/register`
- `POST /api/push/devices/unregister`

Native push registration request:

```json
{
  "installationId": "installation-uuid",
  "provider": "fcm",
  "platform": "android",
  "token": "push-token",
  "appVersion": "1.0.0",
  "deviceName": "Pixel 9"
}
```

## 5. Realtime contracts

## 5.1 Session / presence

Client -> server:

- `register(userId)`
- `presence:get(ack)`
- `call:pending:get({ userId? })`

Server -> client:

- `user:online` `{ "userId": "uuid" }`
- `user:offline` `{ "userId": "uuid" }`

`presence:get` использует ack callback со списком `string[]`.

## 5.2 Chat events

Client -> server:

- `typing:start` `{ "chatId": "chat-id", "userName": "Alice" }`
- `typing:stop` `{ "chatId": "chat-id" }`
- `messages:read` `{ "chatId": "chat-id" }`

Server -> client:

- `typing:start:{userId}` `{ "chatId": "chat-id", "userId": "uuid", "userName": "Alice" }`
- `typing:stop:{userId}` `{ "chatId": "chat-id", "userId": "uuid" }`
- `message:new:{userId}` `{ "chatId": "chat-id", "message": <Message DTO> }`
- `message:read:{userId}` `{ "chatId": "chat-id", "messageIds": ["m1"], "readAt": "ISO" }`
- `message:edited:{userId}` `{ "chatId": "chat-id", "messageId": "m1", "text": "...", "iv": "...", "isEdited": true }`
- `message:deleted:{userId}` `{ "chatId": "chat-id", "messageId": "m1" }`
- `reaction:updated:{userId}` `{ "chatId": "chat-id", "messageId": "m1", "reactions": [...] }`
- `chat:created:{userId}` `<Chat Summary DTO>`
- `chat:updated:{userId}` `{ "chatId": "chat-id" }`
- `chat:members_updated:{userId}` `{ "chatId": "chat-id" }`
- `chat:member_added:{userId}` `{ "chatId": "chat-id", "userId": "member-id" }`
- `chat:member_left:{userId}` `{ "chatId": "chat-id", "userId": "member-id" }`
- `chat:deleted` `{ "chatId": "chat-id" }`
- `user:blocked:{userId}` `{ "blockerId": "u1", "blockedId": "u2" }`
- `user:unblocked:{userId}` `{ "blockerId": "u1", "blockedId": "u2" }`

## 5.3 Direct call signaling

Client -> server:

- `call:initiate` `{ "to": "user-id", "from": "user-id", "offer": RTCSessionDescriptionInit, "callType": "audio" | "video", "fromName": "Alice", "fromAvatar": "🙂" }`
- `call:answer` `{ "to": "user-id", "from": "user-id", "answer": RTCSessionDescriptionInit }`
- `call:reject` `{ "to": "user-id", "from": "user-id" }`
- `call:end` `{ "to": "user-id", "from": "user-id" }`
- `call:ice-candidate` `{ "to": "user-id", "from": "user-id", "candidate": RTCIceCandidateInit }`
- `call:renegotiate` `{ "to": "user-id", "from": "user-id", "offer": RTCSessionDescriptionInit }`
- `call:renegotiate-answer` `{ "to": "user-id", "from": "user-id", "answer": RTCSessionDescriptionInit }`

Server -> client:

- `call:incoming` `{ "from": "user-id", "offer": RTCSessionDescriptionInit, "callType": "audio" | "video", "fromName": "Alice", "fromAvatar": "🙂" }`
- `call:answered` `{ "from": "user-id", "answer": RTCSessionDescriptionInit }`
- `call:rejected` `{ "from": "user-id" }`
- `call:ended` `{ "from": "user-id" }`
- `call:ice-candidate` `{ "from": "user-id", "candidate": RTCIceCandidateInit }`
- `call:renegotiate` `{ "from": "user-id", "offer": RTCSessionDescriptionInit }`
- `call:renegotiate-answer` `{ "from": "user-id", "answer": RTCSessionDescriptionInit }`
- `call:error` `{ "message": "Call was not answered" }`

Сервер теперь предпочитает user id из socket bearer token. Поле `from` оставлено для совместимости со старым web flow.

## 5.4 Group call signaling

Client -> server:

- `group-call:join` `{ "groupId": "chat-id", "userId": "user-id", "userName": "Alice", "userAvatar": "🙂" }`
- `group-call:leave` `{ "groupId": "chat-id", "userId": "user-id" }`
- `group-call:offer` `{ "groupId": "chat-id", "to": "user-id", "from": "user-id", "offer": RTCSessionDescriptionInit }`
- `group-call:answer` `{ "groupId": "chat-id", "to": "user-id", "from": "user-id", "answer": RTCSessionDescriptionInit }`
- `group-call:ice-candidate` `{ "groupId": "chat-id", "to": "user-id", "from": "user-id", "candidate": RTCIceCandidateInit }`
- `group-call:peer-state` `{ "groupId": "chat-id", "userId": "user-id", "isScreenSharing": true, "isMuted": false }`

Server -> client:

- `group-call:room-info` `{ "groupId": "chat-id", "participants": [{ "userId": "user-id", "name": "Alice", "avatar": "🙂", "isScreenSharing": false, "isMuted": false }] }`
- `group-call:peer-joined` `{ "groupId": "chat-id", "userId": "user-id", "userName": "Alice", "userAvatar": "🙂" }`
- `group-call:offer` `{ "groupId": "chat-id", "from": "user-id", "offer": RTCSessionDescriptionInit }`
- `group-call:answer` `{ "groupId": "chat-id", "from": "user-id", "answer": RTCSessionDescriptionInit }`
- `group-call:ice-candidate` `{ "groupId": "chat-id", "from": "user-id", "candidate": RTCIceCandidateInit }`
- `group-call:peer-left` `{ "groupId": "chat-id", "userId": "user-id" }`
- `group-call:peer-state` `{ "userId": "user-id", "isScreenSharing": true, "isMuted": false }`

## 6. Найденные риски и текущие допущения

### Уже выровнено

- Исправлен `POST /api/auth/register`: SQL вставка теперь согласована с колонкой `quick_reaction_emoji`.
- Socket signaling теперь поддерживает bearer token auth на handshake и канонизирует user id из токена, не ломая текущий `register(userId)` flow.

### Остаются важные ограничения

- `mediaUrl`, `previewUrl`, `assetUrl` сервер отдает относительными путями, не абсолютными URL.
- Push payload содержит web-style `url` c query params (`openChat`, `incomingCallerId`). Для native-клиентов лучше опираться на структурные поля `chatId`, `callerId`, `type`, а `url` считать backward-compatible legacy полем.
- Password reset flow строит ссылку на web route (`/?resetToken=...`), это не native-first контракт.
- Chat read receipts сейчас chat-level, а не per-user/per-message receipt model. Для mobile это достаточно для текущего UI, но не для детальной матрицы read state в группах.
- Socket события `message:new:{userId}` и похожие используют user-id в имени события. Это стабильно, но менее удобно для typed mobile SDK, чем единый event name + payload.
- Group/direct call signaling пока не валидирует membership/ACL так глубоко, как REST endpoints.

## 7. Рекомендуемый следующий шаг

- Следующим минимальным шагом имеет смысл вынести эти DTO в shared typed contract layer (`server/src/contracts` или отдельный пакет типов), не меняя transport.
- После этого можно добавить 1 mobile-oriented bootstrap endpoint только если native-клиенту действительно не хватает агрегированного initial sync.
