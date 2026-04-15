# Karas Messenger — Android Architecture

> Версия 1.0 · 2026-04-03  
> Базовая архитектура для отдельного native Android-клиента.

## 1. Контекст и цель

Сейчас в репозитории уже есть:

- `src/` — web-клиент на React/Vite;
- `android/` — существующий Capacitor Android shell;
- `server/` — backend с REST, Socket.IO signaling, push и mobile bootstrap;
- `docs/MOBILE_API_CONTRACTS_RU.md` — минимальный стабильный контракт для native mobile.

Для отдельного native Android-клиента лучше не расширять текущий `android/` shell, а завести новый Android-проект рядом, условно в `android-native/`. Это позволит:

- не тащить web-архитектуру в нативный слой;
- строить полноценный Android lifecycle под realtime и calls;
- отдельно развивать push, foreground service, WebRTC и offline cache.

## 2. Предлагаемый стек

| Слой | Решение | Почему это подходит |
|---|---|---|
| UI | Jetpack Compose | Нативный declarative UI, быстрый старт, хорошо сочетается с screen state и realtime |
| Navigation | Navigation Compose для main flow + отдельный `CallActivity` для звонков | Основной app flow остаётся простым, а звонки не зависят от текущего экрана и корректно открываются из full-screen notification |
| State management | `ViewModel` + `StateFlow` + UDF по экранам | Предсказуемая модель для messenger UI, без тяжёлого внешнего MVI/TCA-подобного фреймворка |
| DI | Hilt | Стандартно для Android, удобно для модулей, сервисов и ViewModel |
| Networking | Retrofit + OkHttp + Kotlinx Serialization | Типизированный REST, interceptors, auth, upload и хорошая совместимость с текущим API |
| Realtime / Socket | `socket.io-client-java` в собственной обёртке `SocketService` | Сервер уже использует Socket.IO, а не чистый WebSocket |
| Calls / media | `org.webrtc:google-webrtc` + foreground service | Подходит под direct/group calls и screen share lifecycle |
| Storage | Room + Proto DataStore + `SecureStore` поверх Android Keystore | Room для chat cache, DataStore для флагов/настроек, Keystore для JWT и E2EE ключей |
| Push | Firebase Cloud Messaging | Бэкенд уже поддерживает `POST /api/push/devices/register` с `provider: "fcm"` |
| Crypto | Kotlin/JCA + Android Keystore | Нужно повторить текущую модель E2EE: ECDH + AES-GCM, private key хранить отдельно от Room |

## 3. Ключевые архитектурные решения

### 3.1 App model

- Основной режим: single-activity app на Compose.
- `MainActivity` содержит `NavHost` для auth/main/settings/profile.
- Для входящего и активного звонка используется отдельный `CallActivity`, который привязан к `CallService`.
- Источник истины для UI: локальная база (`Room`) + in-memory session state.

Такой гибрид лучше, чем "всё через один NavHost", потому что call UI живёт дольше обычного экрана, должен подниматься из background и не должен ломаться при смене текущего route.

### 3.2 Navigation model

Рекомендуемая схема:

```text
Root
├── Splash
├── AuthGraph
│   ├── Login
│   └── Register
└── MainGraph
    ├── ChatList
    ├── Conversation(chatId)
    ├── ChatInfo(chatId)
    ├── Profile(userId)
    └── Settings

Call flow:
- Incoming call notification -> CallActivity
- Active call -> CallActivity
- Group call -> CallActivity(groupId)
```

Что важно:

- звонок не должен быть обычным экраном в back stack чатов;
- deep link/open-from-push должен уметь открыть либо `Conversation`, либо `CallActivity`;
- навигация по main flow должна быть typed route based, без ручной string-склейки по всему проекту.

### 3.3 State management

Подход: MVVM + screen-level UDF.

На каждый feature screen:

- `UiState` — то, что рендерит Compose;
- `UiAction` — действия пользователя;
- `UiEffect` — одноразовые события: toast, navigation, permission prompt;
- `ViewModel` — orchestration;
- `Repository` — данные и sync;
- `UseCase` — только там, где реально есть логика, а не ради слоя.

Почему не брать тяжёлый MVI framework:

- для мессенджера важнее скорость итераций и контроль над lifecycle;
- большая часть сложности здесь в sync, sockets, calls и storage, а не в reducer machinery;
- `ViewModel` + `StateFlow` уже достаточно хорошо покрывают screen state.

### 3.4 Networking layer

Базовый набор:

- `Retrofit` для REST;
- `OkHttp` для auth interceptor, retry policy и upload;
- `Kotlinx Serialization` для DTO;
- единый `ServerUrlResolver` для преобразования относительных `mediaUrl`/`assetUrl` в абсолютные URL.

Особенности под Karas:

- использовать `GET /api/mobile/bootstrap` как основной initial sync;
- bearer token передавать в `Authorization`;
- upload идти отдельным multipart endpoint `POST /api/upload`;
- `clientRequestId` обязателен для optimistic sending и дедупликации.

### 3.5 Realtime / socket layer

Нужен отдельный `SocketService`, а не прямые вызовы socket из ViewModel.

Задачи `SocketService`:

- подключение по `auth.token = "Bearer <jwt>"`;
- reconnect/backoff;
- повторный `call:pending:get` после reconnect;
- `presence:get` после успешного connect;
- нормализация динамических событий вида `message:new:{userId}` в typed internal events;
- разделение chat events и call signaling.

Рекомендуемая модель:

- Socket принимает raw events.
- Mapper преобразует их в `SocketEvent`.
- Sync layer применяет события к Room.
- UI наблюдает только `Flow` из Room/Repository.

Это важно: UI не должен напрямую "жить на сокете". Иначе будет сложно переживать reconnect, process death и параллельный update из REST/bootstrap.

### 3.6 Calls / realtime media

Нужно разделить:

- signaling — через Socket.IO;
- media transport — через WebRTC;
- call lifecycle — через foreground service;
- UI — через `CallActivity`.

Рекомендуемые компоненты:

- `CallSessionManager`
- `WebRtcPeerFactory`
- `DirectCallCoordinator`
- `GroupCallCoordinator`
- `CallService`

На первом этапе не обязательно сразу интегрировать `ConnectionService` и системный dialer UI. Для MVP достаточно:

- full-screen incoming call notification;
- foreground service во время активного звонка;
- audio/video/direct/group call screen;
- screen share как phase 2.

### 3.7 Storage layer

Разделение по типам данных:

- `Room`: chats, messages, participants, reactions, call history, sync markers;
- `Proto DataStore`: theme, notification prefs, draft flags, onboarding flags;
- `SecureStore`: JWT, installationId, encryptedPrivateKey, decrypted session key material;
- файловый cache: media thumbnails, временные upload/download файлы.

Критично:

- JWT и E2EE private key не хранить в Room;
- сообщения можно кешировать в Room, но криптографический материал должен жить отдельно;
- при bootstrap сначала поднимать secure session, затем sync.

## 4. Модульная структура

Не стоит начинать с 25 Gradle-модулей. Для Karas достаточно умеренной modularization, которую легко расширить.

```text
android-native/
├── app
├── core
│   ├── common
│   ├── model
│   ├── designsystem
│   ├── ui
│   ├── navigation
│   ├── network
│   ├── socket
│   ├── database
│   ├── datastore
│   ├── crypto
│   ├── notifications
│   └── webrtc
├── data
│   ├── auth
│   ├── chats
│   └── calls
└── feature
    ├── auth
    ├── chatlist
    ├── conversation
    ├── chatinfo
    ├── profile
    ├── settings
    └── call
```

### Назначение модулей

`app`

- application class;
- root DI graph;
- activities;
- startup orchestration;
- root navigation.

`core:model`

- shared domain models;
- enums;
- typed IDs;
- общие UI-independent модели.

`core:network`

- Retrofit services;
- auth interceptor;
- serialization config;
- URL resolver;
- error mapping.

`core:socket`

- Socket.IO client;
- event parser;
- typed socket events;
- reconnect policy.

`core:database`

- Room database;
- entities;
- DAO;
- migrations.

`core:crypto`

- E2EE encrypt/decrypt;
- key import/export;
- secure key storage bridge.

`core:webrtc`

- peer connection factory;
- audio/video track management;
- ICE/SDP adapters.

`data:auth`

- auth repository;
- session repository;
- bootstrap repository.

`data:chats`

- chat list sync;
- conversation sync;
- message send/edit/delete/reaction;
- read receipts;
- typing.

`data:calls`

- direct and group call repositories/coordinators;
- call signaling orchestration;
- call state sync with service.

`feature:*`

- screen UI;
- ViewModel;
- feature-specific mappers.

## 5. Модели

### 5.1 DTO

DTO должны почти 1-в-1 повторять серверный контракт и не содержать UI-логики.

```kotlin
@Serializable
data class BootstrapDto(
    val contractsVersion: String,
    val serverTime: String,
    val user: UserDto,
    val e2ee: E2eeStateDto,
    val chats: List<ChatSummaryDto>,
    val realtime: RealtimeConfigDto,
)

@Serializable
data class UserDto(
    val id: String,
    val username: String? = null,
    val name: String,
    val displayName: String? = null,
    val avatar: String,
    val bio: String? = null,
    val phone: String? = null,
    val email: String? = null,
    val quickReactionEmoji: String? = null,
    val hasPassword: Boolean? = null,
    val hasKeyBackup: Boolean? = null,
    val authProvider: String? = null,
    val keyBackupType: String? = null,
    val needsE2eeBackupSetup: Boolean? = null,
    val hasE2eePublicKey: Boolean? = null,
)

@Serializable
data class ChatSummaryDto(
    val id: String,
    val isGroup: Boolean,
    val isChannel: Boolean,
    val chatType: String,
    val name: String,
    val avatar: String,
    val otherMembers: List<ChatMemberDto> = emptyList(),
    val otherUserIds: List<String> = emptyList(),
    val lastMessage: LastMessageDto? = null,
    val unreadCount: Int,
    val membersCount: Int? = null,
    val allowSelfSubscribe: Boolean = false,
    val createdAt: String,
    val lastSeenAt: String? = null,
    val username: String? = null,
    val bio: String? = null,
    val phone: String? = null,
    val isBlockedByMe: Boolean = false,
    val hasBlockedMe: Boolean = false,
    val isAdmin: Boolean = false,
    val isOwner: Boolean = false,
    val isPinned: Boolean = false,
    val notificationsEnabled: Boolean = true,
)

@Serializable
data class MessageDto(
    val id: String,
    val senderId: String,
    val clientRequestId: String? = null,
    val replyToMessageId: String? = null,
    val contentType: String,
    val text: String? = null,
    val iv: String? = null,
    val mediaUrl: String? = null,
    val mediaType: String? = null,
    val mediaName: String? = null,
    val sticker: StickerDto? = null,
    val createdAt: String,
    val read: Boolean,
    val readAt: String? = null,
    val isEdited: Boolean,
    val senderName: String,
    val senderAvatar: String,
    val isMine: Boolean,
    val reactions: List<ReactionDto> = emptyList(),
)
```

### 5.2 Domain models

Domain слой скрывает transport details и даёт более удобные типы для бизнес-логики.

```kotlin
data class User(
    val id: UserId,
    val displayName: String,
    val username: String?,
    val avatar: AvatarRef,
    val bio: String?,
    val phone: String?,
    val hasE2eeKey: Boolean,
)

data class Chat(
    val id: ChatId,
    val title: String,
    val avatar: AvatarRef,
    val type: ChatType,
    val members: List<UserPreview>,
    val unreadCount: Int,
    val lastMessage: MessagePreview?,
    val permissions: ChatPermissions,
    val flags: ChatFlags,
)

data class Message(
    val id: MessageId,
    val chatId: ChatId,
    val senderId: UserId,
    val content: MessageContent,
    val createdAt: Instant,
    val status: MessageStatus,
    val isEdited: Boolean,
    val replyToMessageId: MessageId?,
    val reactions: List<MessageReaction>,
)

sealed interface MessageContent {
    data class Text(val body: String, val isEncrypted: Boolean) : MessageContent
    data class Media(val url: String, val type: MediaType, val name: String?) : MessageContent
    data class Sticker(val stickerId: String, val packId: String) : MessageContent
}

data class CallSession(
    val id: String,
    val scope: CallScope,
    val state: CallState,
    val participants: List<CallParticipant>,
    val isMuted: Boolean,
    val isSpeakerOn: Boolean,
    val isScreenSharing: Boolean,
)
```

### 5.3 UI state

UI state должен быть плоским, serializable по смыслу и не тащить в себя DTO.

```kotlin
data class ChatListUiState(
    val isLoading: Boolean = false,
    val isRefreshing: Boolean = false,
    val query: String = "",
    val chats: List<ChatListItemUi> = emptyList(),
    val socketConnected: Boolean = false,
    val error: UiText? = null,
)

data class ConversationUiState(
    val chatId: String,
    val title: String = "",
    val messages: List<MessageUi> = emptyList(),
    val draft: String = "",
    val isSending: Boolean = false,
    val typingUsers: List<String> = emptyList(),
    val canLoadMore: Boolean = true,
    val attachmentMenuVisible: Boolean = false,
    val error: UiText? = null,
)

data class CallUiState(
    val callId: String? = null,
    val mode: CallMode = CallMode.Audio,
    val phase: CallPhase = CallPhase.Idle,
    val title: String = "",
    val participants: List<CallParticipantUi> = emptyList(),
    val localVideoEnabled: Boolean = false,
    val localAudioEnabled: Boolean = true,
    val isConnecting: Boolean = false,
    val error: UiText? = null,
)
```

## 6. Потоки данных

### 6.1 App startup

1. `SessionManager` читает JWT и secure key material из `SecureStore`.
2. Если JWT нет, приложение идёт в `AuthGraph`.
3. Если JWT есть, вызывается `GET /api/mobile/bootstrap`.
4. Ответ сохраняется в Room и secure storage.
5. После bootstrap поднимается `SocketService`.
6. После connect выполняются:
   - `call:pending:get`
   - `presence:get`
   - push device registration при необходимости
7. UI читает данные из repositories/Room.

### 6.2 Conversation sync

1. `ConversationViewModel` подписан на `messagesFlow(chatId)`.
2. `ChatRepository` отдаёт данные из Room.
3. Socket event `message:new:{userId}` обновляет Room.
4. UI автоматически перерисовывается.
5. При отправке сообщения создаётся optimistic local message с `clientRequestId`.
6. После REST success локальная запись merge-ится с серверной.

### 6.3 Call flow

1. Пользователь инициирует звонок.
2. `CallSessionManager` создаёт локальную session.
3. `SocketService` отправляет `call:initiate` или `group-call:join`.
4. `CallService` переходит в foreground.
5. `WebRtcPeerFactory` создаёт peer connection.
6. ICE/SDP проходят через `SocketService`.
7. `CallActivity` отображает состояние из `CallRepository`.

## 7. Практический scaffold для Android Studio

Следующим шагом в Android Studio стоит создавать не UI-экраны целиком, а каркас приложения.

### Шаг 1. Создать новый проект

- новый проект `android-native/`;
- Kotlin + Compose;
- single activity template;
- Gradle Kotlin DSL;
- package: `app.karas.messenger.nativeapp` или короче `app.karas.native`.

### Шаг 2. Сразу завести модули

Минимально на старте:

- `:app`
- `:core:model`
- `:core:network`
- `:core:socket`
- `:core:database`
- `:core:crypto`
- `:core:webrtc`
- `:data:auth`
- `:data:chats`
- `:data:calls`
- `:feature:auth`
- `:feature:chatlist`
- `:feature:conversation`
- `:feature:call`

Остальное можно вынести позже.

### Шаг 3. Сначала создать базовые артефакты

- `KarasApplication`
- `MainActivity`
- `CallActivity`
- `AppNavHost`
- `SessionManager`
- `BootstrapApi`
- `SocketService`
- `AppDatabase`
- `SecureStore`
- `CallService`
- `ChatRepository`
- `CallRepository`

### Шаг 4. Первый вертикальный slice

Первый рабочий slice лучше делать таким:

1. Login
2. `GET /api/mobile/bootstrap`
3. Chat list из Room
4. Socket connect
5. Incoming `message:new`
6. Conversation screen без сложного media UI

После этого уже подключать:

- typing/read receipts;
- direct calls;
- group calls;
- push;
- screen share;
- stickers и расширенные media flows.

## 8. Итоговая рекомендация

Для Karas Messenger оптимальна архитектура:

- Compose UI;
- Navigation Compose для main flows;
- отдельный `CallActivity` + foreground service для calls;
- `ViewModel` + `StateFlow` + UDF;
- Retrofit/OkHttp для REST;
- `socket.io-client-java` в typed socket wrapper;
- Room как source of truth;
- DataStore для prefs;
- Keystore-backed secure storage для JWT и E2EE;
- выделенный call слой поверх WebRTC.

Это даст Android-клиенту:

- нормальный realtime lifecycle;
- корректную модель для звонков и background/incoming call;
- устойчивый offline/read-model;
- понятную и расширяемую модульную структуру без преждевременного переусложнения.
