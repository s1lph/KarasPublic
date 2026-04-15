# Karas Messenger — iOS Architecture

> Версия 1.0 · 2026-04-03  
> Эталонный документ для разработки нативного iOS-клиента.

---

## 1. Технологический стек

| Слой | Решение | Обоснование |
|---|---|---|
| **UI** | SwiftUI | Нативно, декларативно; UIKit только для WebRTC view и кастомного ввода |
| **Navigation** | `NavigationStack` + `NavigationPath` (iOS 16+) | Программная навигация без UIKit-костылей |
| **State management** | `@Observable` / `ObservableObject` + `EnvironmentObject` | Без сторонних фреймворков; TCA избыточен для этого масштаба |
| **Networking** | `URLSession` + `async/await` + `Codable` | Стандартно, без зависимостей |
| **Realtime / Socket** | `socket.io-client-swift` (SPM) | Сервер использует Socket.IO; нативный WebSocket несовместим с polling-fallback |
| **WebRTC** | `GoogleWebRTC` / `WebRTC.xcframework` | Сигнализация готова на сервере (`call:initiate`, `call:answer`, ICE) |
| **Storage** | `SwiftData` (iOS 17+) | Кеш чатов и сообщений; Keychain — для JWT и E2EE ключей |
| **Push** | APNs + `UNUserNotificationCenter` | Бэкенд поддерживает `POST /push/devices/register` с `provider: "apns"` |
| **E2EE** | `CryptoKit` | Тот же алгоритм (ECDH + AES-GCM), что и в веб-клиенте |

---

## 2. Ключевые находки при анализе бэкенда

- **`GET /api/mobile/bootstrap`** — единственный запрос при старте: возвращает `user`, все `chats`, E2EE-состояние и realtime-конфиги.
- **APNs полностью поддержан** — `POST /api/push/devices/register` принимает `{ provider: "apns", platform: "ios", token, installationId }`.
- **Socket.IO JWT-авторизация** — передаётся через `auth: { token: "Bearer <jwt>" }` при подключении; отдельный `register`-эвент не обязателен.
- **Сигнализация звонков** — `call:initiate → call:answer → call:ice-candidate → call:end`. Групповые звонки: `group-call:join/leave/offer/answer/ice-candidate`.
- **Typing** — `typing:start` / `typing:stop` с `chatId`.
- **Presence** — `user:online` / `user:offline` широковещательно.
- **Read receipts** — `messages:read { chatId }` на сокете.
- **E2EE** — `iv` хранится в `MessageDTO`; закрытый ключ зашифрован на сервере (`encryptedPrivateKey` + `keySalt`).

---

## 3. Структура проекта

```
messenger/
├── server/          # Node.js бэкенд
├── src/             # React веб-клиент
├── android/         # Capacitor Android
└── ios/             # Нативный iOS-клиент
    └── KarasMessenger/
        ├── App/
        │   ├── KarasApp.swift              # @main, DI
        │   └── AppRouter.swift             # Глобальный стейт навигации + auth флаг
        ├── Core/
        │   ├── Network/
        │   │   ├── APIClient.swift         # URLSession + async/await
        │   │   ├── Endpoint.swift          # Enum всех эндпоинтов
        │   │   └── NetworkError.swift
        │   ├── Socket/
        │   │   ├── SocketManager.swift     # socket.io-client-swift обёртка
        │   │   └── SocketEvents.swift      # Enum всех событий
        │   ├── Storage/
        │   │   ├── PersistenceController.swift   # SwiftData
        │   │   └── Models+SwiftData.swift
        │   ├── Keychain/
        │   │   └── KeychainService.swift   # JWT токен, E2EE ключи
        │   ├── Push/
        │   │   ├── PushNotificationHandler.swift
        │   │   └── DeviceTokenManager.swift
        │   └── Crypto/
        │       └── E2EEService.swift       # CryptoKit ECDH + AES-GCM
        ├── Models/
        │   ├── DTO/                        # Строго Codable, server contract
        │   │   ├── AuthDTO.swift
        │   │   ├── BootstrapDTO.swift
        │   │   ├── ChatDTO.swift
        │   │   └── MessageDTO.swift
        │   ├── Domain/                     # Бизнес-сущности
        │   │   ├── User.swift
        │   │   ├── Chat.swift
        │   │   └── Message.swift
        │   └── UIState/                    # ViewModel state structs
        │       ├── ChatListState.swift
        │       ├── ConversationState.swift
        │       └── CallState.swift
        ├── Features/
        │   ├── Auth/
        │   │   ├── LoginView.swift
        │   │   ├── RegisterView.swift
        │   │   └── AuthViewModel.swift
        │   ├── ChatList/
        │   │   ├── ChatListView.swift
        │   │   ├── ChatRowView.swift
        │   │   └── ChatListViewModel.swift
        │   ├── Conversation/
        │   │   ├── ConversationView.swift
        │   │   ├── MessageBubbleView.swift
        │   │   ├── InputBarView.swift
        │   │   ├── MediaViewerView.swift
        │   │   └── ConversationViewModel.swift
        │   ├── Calls/
        │   │   ├── CallView.swift
        │   │   ├── IncomingCallView.swift
        │   │   ├── WebRTCService.swift
        │   │   └── CallViewModel.swift
        │   ├── Profile/
        │   │   ├── MyProfileView.swift
        │   │   └── ProfileViewModel.swift
        │   ├── UserProfile/
        │   │   └── UserProfileView.swift
        │   ├── Groups/
        │   │   ├── CreateGroupView.swift
        │   │   └── GroupSettingsView.swift
        │   └── Stickers/
        │       ├── StickerPickerView.swift
        │       └── StickerPackView.swift
        └── Shared/
            ├── UI/
            │   ├── AvatarView.swift
            │   ├── LoadingView.swift
            │   └── ErrorBannerView.swift
            ├── Extensions/
            │   ├── JSONDecoder+ISO8601.swift
            │   └── View+Modifiers.swift
            └── Utils/
                └── DateFormatter+Relative.swift
```

---

## 4. Модели

### 4.1 DTO (Codable, server contract)

```swift
struct BootstrapDTO: Codable {
    let contractsVersion: String
    let serverTime: String
    let user: UserDTO
    let e2ee: E2eeStateDTO
    let chats: [ChatDTO]
    let realtime: RealtimeInfoDTO
}

struct UserDTO: Codable {
    let id: String
    let name: String
    let username: String?
    let displayName: String?
    let avatar: String          // emoji или относительный URL
    let bio: String?
    let online: Bool?
    let lastSeen: String?       // ISO8601
    let hasE2eePublicKey: Bool?
}

struct ChatDTO: Codable {
    let id: String
    let name: String
    let avatar: String
    let isGroup: Bool
    let isChannel: Bool?
    let lastMessage: LastMessageDTO?
    let unreadCount: Int
    let isPinned: Bool
    let notificationsEnabled: Bool
    let isAdmin: Bool?
    let isOwner: Bool?
    let isBlockedByMe: Bool?
    let hasBlockedMe: Bool?
    let membersCount: Int?
}

struct MessageDTO: Codable {
    let id: String
    let senderId: String
    let clientRequestId: String?
    let replyToMessageId: String?
    let contentType: String         // "text" | "media" | "sticker"
    let text: String?
    let iv: String?                 // E2EE initialization vector
    let mediaUrl: String?
    let mediaType: String?          // "image" | "audio" | "video" | "file"
    let mediaName: String?
    let sticker: StickerPayloadDTO?
    let createdAt: String
    let read: Bool
    let isEdited: Bool
    let senderName: String
    let senderAvatar: String
    let isMine: Bool
}

struct AuthResponseDTO: Codable {
    let token: String
    let user: UserDTO
    let encryptedPrivateKey: String?
    let keySalt: String?
    let hasKeyBackup: Bool
    let keyBackupType: String       // "account_password" | "separate_secret"
    let hasKeys: Bool
}
```

### 4.2 Domain models

```swift
struct User: Identifiable, Hashable {
    let id: String
    var displayName: String
    var username: String?
    var avatar: String
    var bio: String?
    var isOnline: Bool
    var lastSeen: Date?
    var hasE2eeKey: Bool
}

struct Chat: Identifiable, Hashable {
    let id: String
    var name: String
    var avatar: String
    var type: ChatType              // .direct | .group | .channel
    var lastMessage: LastMessage?
    var unreadCount: Int
    var isPinned: Bool
    var isMuted: Bool
    var isBlockedByMe: Bool
    var hasBlockedMe: Bool
}

struct Message: Identifiable, Hashable {
    let id: String
    let chatId: String
    let senderId: String
    var content: MessageContent
    let timestamp: Date
    var isRead: Bool
    var isEdited: Bool
    var isMine: Bool
    var replyToId: String?
}

enum ChatType      { case direct, group, channel }
enum MessageContent {
    case text(String, encrypted: Bool)
    case media(url: URL, type: MediaType, name: String?)
    case sticker(StickerPayloadDTO)
}
enum MediaType     { case image, audio, video, file }
```

### 4.3 UI State (ViewModels)

```swift
@Observable class ChatListViewModel {
    var chats: [Chat] = []
    var searchQuery: String = ""
    var isLoading = false
    var error: AppError? = nil

    var pinned: [Chat]   { chats.filter(\.isPinned) }
    var unpinned: [Chat] { chats.filter { !$0.isPinned } }
}

@Observable class ConversationViewModel {
    var messages: [Message] = []
    var draftText: String = ""
    var typingUsers: [String: Bool] = [:]   // userId → isTyping
    var replyTo: Message? = nil
    var isLoading = false
    var isSending = false
}

@Observable class CallViewModel {
    var status: CallStatus = .idle
    var callType: CallType = .audio
    var remoteUser: User? = nil
    var isMuted = false
    var isVideoEnabled = false
}

enum CallStatus { case idle, outgoing, incoming, connecting, active, ended }
enum CallType   { case audio, video }
```

---

## 5. Сетевой слой

```swift
// Базовый клиент
class APIClient {
    static let shared = APIClient()
    private let baseURL = URL(string: "https://your-server.com/api")!

    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
        var req = URLRequest(url: baseURL.appending(path: endpoint.path))
        req.httpMethod = endpoint.method
        req.setValue("application/json", forHTTPHeaderField: "Content-Type")
        if let token = KeychainService.shared.token {
            req.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }
        if let body = endpoint.encodedBody { req.httpBody = body }
        let (data, response) = try await URLSession.shared.data(for: req)
        guard (response as? HTTPURLResponse)?.statusCode ?? 0 < 300 else {
            throw NetworkError.serverError(data)
        }
        return try JSONDecoder.iso8601.decode(T.self, from: data)
    }
}

// Ключевые эндпоинты
// GET  /api/mobile/bootstrap         → BootstrapDTO
// POST /api/auth/login               → AuthResponseDTO
// POST /api/auth/register            → AuthResponseDTO
// GET  /api/chats/:id/messages       → [MessageDTO]
// POST /api/chats/:id/messages       → MessageDTO
// PUT  /api/messages/:id             → MessageDTO  (edit)
// DELETE /api/messages/:id           → success
// POST /api/chats                    → ChatDTO  (create direct)
// POST /api/groups                   → ChatDTO  (create group)
// POST /api/upload                   → { mediaUrl, mediaType, originalName }
// POST /api/push/devices/register    → success
// POST /api/messages/:id/reactions   → success
```

---

## 6. Socket Layer

### Подключение
```swift
// socket.io-client-swift
let socket = SocketIOClient(
    socketURL: URL(string: serverURL)!,
    config: [.connectParams(["auth": ["token": "Bearer \(jwt)"]])]
)
socket.connect()
```

### События сервер → клиент
| Событие | Payload | Действие |
|---|---|---|
| `message:new` | `MessageDTO + chatId` | Добавить в ConversationVM |
| `message:edited` | `{ messageId, text, iv }` | Обновить сообщение |
| `message:deleted` | `{ messageId, chatId }` | Удалить из списка |
| `messages:read` | `{ chatId, messageIds, readAt }` | Обновить read-статус |
| `user:online` | `{ userId }` | Обновить presence |
| `user:offline` | `{ userId }` | Обновить presence |
| `typing:start:userId` | `{ chatId, userId, userName }` | Показать typing indicator |
| `typing:stop:userId` | `{ chatId, userId }` | Скрыть typing indicator |
| `call:incoming` | `{ from, offer, callType, fromName, fromAvatar }` | CallKit incoming call |
| `call:answered` | `{ from, answer }` | WebRTC answer |
| `call:rejected` | `{ from }` | Показать "отклонено" |
| `call:ended` | `{ from }` | Завершить звонок |
| `call:ice-candidate` | `{ from, candidate }` | ICE candidate |

### События клиент → сервер
| Событие | Payload |
|---|---|
| `typing:start` | `{ chatId, userName }` |
| `typing:stop` | `{ chatId }` |
| `messages:read` | `{ chatId }` |
| `call:initiate` | `{ to, from, offer, callType, fromName, fromAvatar }` |
| `call:answer` | `{ to, from, answer }` |
| `call:reject` | `{ to, from }` |
| `call:end` | `{ to, from }` |
| `call:ice-candidate` | `{ to, from, candidate }` |

---

## 7. Push Notifications (APNs)

```swift
// 1. Запросить разрешение
UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .sound, .badge])

// 2. Зарегистрировать APNs токен
UIApplication.shared.registerForRemoteNotifications()

// 3. Отправить токен на сервер
func application(_ app: UIApplication, didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
    let token = deviceToken.map { String(format: "%02x", $0) }.joined()
    Task {
        try await APIClient.shared.registerPushDevice(PushDevicePayload(
            installationId: UIDevice.current.identifierForVendor!.uuidString,
            provider: "apns",
            platform: "ios",
            token: token,
            appVersion: Bundle.main.shortVersionString,
            deviceName: UIDevice.current.name
        ))
    }
}

// 4. Входящий звонок → CallKit
// push payload type == "call:incoming" → reportNewIncomingCall(with: CXCallUpdate)
```

---

## 8. E2EE (CryptoKit)

```swift
class E2EEService {
    // Разблокировать приватный ключ
    func unlockPrivateKey(encryptedKey: String, salt: String, password: String) throws -> P256.KeyAgreement.PrivateKey

    // Зашифровать сообщение для получателя
    func encrypt(text: String, recipientPublicKey: P256.KeyAgreement.PublicKey) throws -> (ciphertext: String, iv: String)

    // Расшифровать входящее сообщение (iv из MessageDTO)
    func decrypt(ciphertext: String, iv: String, senderPublicKey: P256.KeyAgreement.PublicKey) throws -> String
}
```

---

## 9. Navigation

```swift
@main
struct KarasApp: App {
    @State private var router = AppRouter()

    var body: some Scene {
        WindowGroup {
            if router.isAuthenticated {
                MainView().environment(router)
            } else {
                AuthFlow()
            }
        }
    }
}

struct MainView: View {
    var body: some View {
        NavigationStack {
            ChatListView()
                .navigationDestination(for: Chat.self) { ConversationView(chat: $0) }
                .navigationDestination(for: User.self) { UserProfileView(user: $0) }
        }
    }
}
```

---

## 10. План разработки

| # | Этап | Что создавать |
|---|---|---|
| 1 | **Bootstrap** | Xcode → iOS App (SwiftUI, SwiftData). Bundle ID: `com.karas.messenger`. |
| 2 | **SPM зависимости** | `socket.io-client-swift`, `GoogleWebRTC`. |
| 3 | **Core: Keychain + APIClient** | Token storage, базовый `request<T>`. |
| 4 | **Auth screens** | Login / Register → сохранить токен → перейти в ChatList. |
| 5 | **Bootstrap + ChatList** | `GET /mobile/bootstrap` → список чатов. |
| 6 | **Socket** | Подключение с JWT, `message:new`, presence. |
| 7 | **Conversation** | Загрузка сообщений, отправка текста, typing indicator, read receipts. |
| 8 | **Push (APNs)** | Регистрация токена, уведомления о сообщениях. |
| 9 | **Calls (WebRTC + CallKit)** | P2P audio/video. |
| 10 | **E2EE** | CryptoKit decrypt при показе, encrypt при отправке. |
| 11 | **Stickers, Groups, Channels** | После стабилизации ядра. |
