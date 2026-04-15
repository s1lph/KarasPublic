# Karas Messenger — Контракты звонков (Signaling Contracts)

**Версия:** 1.0  
**Статус:** Действующая спецификация (выведена из кода, апрель 2026)  
**Назначение:** Единый источник истины для Web, Android и будущего iOS клиентов. Все платформы обязаны строго соблюдать этот документ. Расхождение с ним — баг.

---

## Содержание

1. [Общие принципы](#1-общие-принципы)
2. [Подключение и аутентификация](#2-подключение-и-аутентификация)
3. [Стейт-машина прямого звонка](#3-стейт-машина-прямого-звонка)
4. [События прямого звонка](#4-события-прямого-звонка)
5. [Протокол восстановления сессии](#5-протокол-восстановления-сессии)
6. [Константы таймаутов](#6-константы-таймаутов)
7. [Recovery-политика](#7-recovery-политика)
8. [Групповые звонки](#8-групповые-звонки)
9. [Push-уведомления](#9-push-уведомления)
10. [Абстрактные интерфейсы платформ](#10-абстрактные-интерфейсы-платформ)
11. [Известные ограничения](#11-известные-ограничения)

---

## 1. Общие принципы

### 1.1 Роль бэкенда

Бэкенд — **Stateful Router**, не SFU/MCU. Он:
- Хранит сигнальные сессии **в памяти** (рестарт сервера уничтожает активные звонки).
- Маршрутизирует SDP и ICE между клиентами по `userId → socketId`.
- Не трогает медиа-потоки (только JSON-сигналинг).
- Обеспечивает таймауты и grace period для переподключений.

### 1.2 Медиа-топология

- **Прямые звонки (1-to-1):** Peer-to-peer (P2P) через WebRTC. Бэкенд не участвует в медиа.
- **Групповые звонки:** Full-mesh P2P — каждый участник коннектится к каждому. **Ограничение: рекомендуется не более 5–6 участников** из-за bandwidth.

### 1.3 Кодеки

Клиенты должны использовать **VP8** или **H.264 Constrained Baseline Profile** для видео. Избегать H.264 High Profile (аппаратно ускоряется на iOS, но может не согласоваться с Web).

### 1.4 ICE-кандидаты

Допускается **два режима**:
1. **Trickle ICE** — кандидаты отправляются по мере появления через `call:ice-candidate`. *Текущий режим на Web.*
2. **Full ICE** — все кандидаты собираются до отправки SDP (`iceGatheringState === 'complete'`). *Текущий режим на Android.*

Оба режима должны поддерживаться обеими сторонами: кандидаты, пришедшие до установки remote description, **буферизуются** и применяются после.

---

## 2. Подключение и аутентификация

### 2.1 Транспорт

```
Протокол: Socket.IO (WebSocket с Long-Polling fallback)
URL:      <server-origin>/socket.io
```

### 2.2 Способы аутентификации (оба поддерживаются)

**Вариант A — JWT на handshake (предпочтительный для нативных клиентов):**
```json
// При подключении socket.io
{
  "auth": {
    "token": "<jwt>"
  }
}
```
Сервер канонизирует `userId` из токена. Поля `from`/`userId` в payload звонков игнорируются, если токен валиден.

**Вариант B — legacy register (backward-compatible):**
```
socket.emit('register', '<userId>')
```
Отправляется сразу после `connect`. Обязателен, если JWT в handshake не передан.

### 2.3 Жизненный цикл сокета при звонке

При **реконнекте** сокета во время активного звонка клиент обязан:
1. Переподключиться (`socket.connect()`).
2. Отправить `call:session:get` → получить `call:session` → принять решение о восстановлении.
3. **Не** разрушать WebRTC PeerConnection, пока не получен ответ `call:session` с `phase: "active"`.

---

## 3. Стейт-машина прямого звонка

### 3.1 Состояния

| Состояние | Описание |
|-----------|----------|
| `IDLE` | Нет активного звонка |
| `INITIATING` | Создаём SDP offer, ждём ICE gathering |
| `RINGING_OUTGOING` | `call:initiate` отправлен, ждём ответа собеседника |
| `RINGING_INCOMING` | Получен `call:incoming`, показываем экран входящего звонка |
| `CONNECTING` | SDP answer получен, идёт обмен ICE-кандидатами |
| `RESTORING` | Приложение перезапущено / сокет переподключён, восстанавливаем сессию |
| `ACTIVE` | ICE Connected, разговор идёт |
| `ENDED` | Звонок завершён нормально (Local hangup / Remote hangup / Отклонён) |
| `FAILED` | Критическая ошибка WebRTC или сигналинга |

### 3.2 Диаграмма переходов

```
                          ┌──────────┐
                          │   IDLE   │◄──────────────────────────────────┐
                          └────┬─────┘                                   │
                               │                                         │ AUTO_RESET (3s)
                    ┌──────────┴──────────┐                              │
                    │                     │                              │
           initiateCall()         call:incoming (event)          ┌──────┴──────┐
                    │                     │                       │    ENDED    │
                    ▼                     ▼                       │    FAILED   │
             ┌────────────┐      ┌────────────────┐              └──────▲──────┘
             │ INITIATING │      │RINGING_INCOMING│                     │
             └─────┬──────┘      └───────┬────────┘              hangUp() / rejectCall()
                   │ offer ready         │ answerCall()          remote hangup / call:ended
                   ▼                     │ rejectCall() ─────────────────┘
          ┌────────────────┐             │
          │RINGING_OUTGOING│             │
          └───────┬────────┘             │
                  │ call:answered        │
                  └──────────┬───────────┘
                             │
                             ▼
                      ┌────────────┐
                      │ CONNECTING │◄────────────────────────────┐
                      └─────┬──────┘                             │
                            │ ICE Connected                       │
                            ▼                                    │
                       ┌────────┐                                │
                       │ ACTIVE │   ──── recovery ─────────────►┘
                       └────────┘   (teardown + rebuild)
                            
              При рестарте приложения:
              IDLE ──[PersistedCallStore + call:session:get]──► RESTORING
                                                                    │
                                                                    ├─► ACTIVE  (сессия жива)
                                                                    └─► IDLE    (сессия устарела)
```

### 3.3 Правила переходов

| Из | Триггер | В |
|----|---------|---|
| `IDLE` | `initiateCall()` | `INITIATING` |
| `IDLE` | `call:incoming` event | `RINGING_INCOMING` |
| `IDLE` | Восстановление из `PersistedCallStore` | `RESTORING` |
| `INITIATING` | Offer создан | `RINGING_OUTGOING` |
| `RINGING_OUTGOING` | `call:answered` event | `CONNECTING` |
| `RINGING_OUTGOING` | `call:rejected` event | `ENDED` |
| `RINGING_OUTGOING` | `CALL_TIMEOUT` (30с) | `ENDED` |
| `RINGING_OUTGOING` | `hangUp()` | `ENDED` |
| `RINGING_INCOMING` | `answerCall()` | `CONNECTING` |
| `RINGING_INCOMING` | `rejectCall()` | `ENDED` |
| `RINGING_INCOMING` | `call:ended` event | `ENDED` |
| `RINGING_INCOMING` | `CALL_TIMEOUT` (30с) | `ENDED` |
| `CONNECTING` | ICE Connected | `ACTIVE` |
| `CONNECTING` | `CONNECTION_TIMEOUT` (20с) | Recovery → `CONNECTING` |
| `ACTIVE` | `hangUp()` | `ENDED` |
| `ACTIVE` | `call:ended` event | `ENDED` |
| `ACTIVE` | ICE Disconnected + grace (4с) | Recovery → `CONNECTING` |
| `ACTIVE` | ICE Failed (после 8с stable) | Recovery → `CONNECTING` |
| `RESTORING` | `call:session` с `phase: "active"` | Recovery → `CONNECTING` |
| `RESTORING` | `call:session` пустой / устаревший | `IDLE` |
| `ENDED` / `FAILED` | `AUTO_RESET` (3с) | `IDLE` |

---

## 4. События прямого звонка

### 4.1 Клиент → Сервер

---

#### `call:initiate` — Инициация звонка

**Отправляет:** Caller  
**Когда:** После создания SDP offer (ICE gathering завершён или начат trickle)

```json
{
  "callId":     "uuid-v4",
  "to":         "userId-callee",
  "from":       "userId-caller",
  "offer":      "<RTCSessionDescriptionInit>",
  "callType":   "audio" | "video",
  "fromName":   "Alice",
  "fromAvatar": "emoji-or-url",
  "toName":     "Bob",
  "toAvatar":   "emoji-or-url"
}
```

**Серверная логика:**
1. Нормализует `callId` (trim whitespace).
2. Очищает устаревшие сессии для обоих участников (caller offline, unanswered).
3. Если один из участников уже в активном звонке — возвращает `call:error`.
4. Создаёт `DirectCallSession` в памяти.
5. Запускает 30-секундный таймер "no answer".
6. Если callee online → `call:incoming`; если offline → push-уведомление.

---

#### `call:answer` — Ответ на звонок

**Отправляет:** Callee  
**Когда:** Пользователь нажал "Принять"

```json
{
  "callId":  "uuid-v4",
  "to":      "userId-caller",
  "from":    "userId-callee",
  "answer":  "<RTCSessionDescriptionInit>"
}
```

**Серверная логика:**
1. Проверяет существование сессии.
2. Записывает `answer` и `answeredAt` в сессию.
3. Отменяет 30-секундный таймер.
4. Роутит ответ на Caller через `call:answered`.

---

#### `call:reject` — Отклонение звонка

**Отправляет:** Callee  
**Когда:** Пользователь нажал "Отклонить" или автоматически по таймеру

```json
{
  "callId": "uuid-v4",
  "to":     "userId-caller",
  "from":   "userId-callee"
}
```

**Серверная логика:** Удаляет сессию из памяти, роутит `call:rejected` на Caller.

---

#### `call:end` — Завершение звонка

**Отправляет:** Любой участник  
**Когда:** Пользователь нажал "Завершить" или звонок закончился по таймеру

```json
{
  "callId": "uuid-v4",
  "to":     "userId-remote",
  "from":   "userId-local"
}
```

**Серверная логика:** Удаляет сессию, отменяет все таймауты, роутит `call:ended` удалённому участнику.

---

#### `call:ice-candidate` — ICE-кандидат

**Отправляет:** Любой участник  
**Когда:** RTCPeerConnection генерирует новый ICE-кандидат (trickle) или после ответа (batch)

```json
{
  "callId":    "uuid-v4",
  "to":        "userId-remote",
  "from":      "userId-local",
  "candidate": {
    "candidate":     "candidate:...",
    "sdpMid":        "0",
    "sdpMLineIndex": 0
  }
}
```

**Серверная логика:** Немедленно роутит кандидат удалённому участнику без буферизации.

**Клиентская обработка (критично):**
- Если `RTCPeerConnection` ещё не создан → **буферизовать** кандидат.
- Если remote description ещё не установлена → **буферизовать** кандидат.
- Применять буфер после `setRemoteDescription()`.

---

#### `call:renegotiate` — Переговоры медиа (mid-call offer)

**Отправляет:** Только **Caller** (инициатор звонка)  
**Когда:** Изменение медиа-треков (screen share on/off) или после ICE recovery

```json
{
  "callId": "uuid-v4",
  "to":     "userId-callee",
  "from":   "userId-caller",
  "offer":  "<RTCSessionDescriptionInit>"
}
```

> ⚠️ **Правило anti-glare:** Только Caller отправляет `call:renegotiate`. Callee всегда отвечает `call:renegotiate-answer`. Это предотвращает "glare" (одновременная отправка offer с обеих сторон).

---

#### `call:renegotiate-answer` — Ответ на переговоры

**Отправляет:** Callee  
**Когда:** В ответ на `call:renegotiate`

```json
{
  "callId": "uuid-v4",
  "to":     "userId-caller",
  "from":   "userId-callee",
  "answer": "<RTCSessionDescriptionInit>"
}
```

---

#### `call:session:get` — Запрос текущей сессии

**Отправляет:** Любой участник  
**Когда:** При переподключении сокета (реконнект, рестарт приложения)

```json
{
  "userId": "userId-self"
}
```

**Серверная логика:** Ищет живую сессию для `userId`, возвращает `call:session` (или пустой объект, если сессии нет).

---

#### `call:pending:get` — Алиас для `call:session:get`

Идентичен `call:session:get`. Существует как backward-compatible дубль. **Новый код должен использовать `call:session:get`.**

---

### 4.2 Сервер → Клиент

---

#### `call:incoming` — Входящий звонок

**Получает:** Callee  
**Когда:** Caller инициировал звонок и Callee онлайн

```json
{
  "callId":    "uuid-v4",
  "from":      "userId-caller",
  "offer":     "<RTCSessionDescriptionInit>",
  "callType":  "audio" | "video",
  "fromName":  "Alice",
  "fromAvatar": "emoji-or-url",
  "createdAt": "2026-04-14T12:00:00.000Z"
}
```

---

#### `call:answered` — Звонок принят

**Получает:** Caller  
**Когда:** Callee ответил

```json
{
  "callId": "uuid-v4",
  "from":   "userId-callee",
  "answer": "<RTCSessionDescriptionInit>"
}
```

---

#### `call:rejected` — Звонок отклонён

**Получает:** Caller

```json
{
  "callId": "uuid-v4",
  "from":   "userId-callee"
}
```

---

#### `call:ended` — Звонок завершён

**Получает:** Удалённый участник

```json
{
  "callId": "uuid-v4",
  "from":   "userId-who-ended",
  "reason": "connection-lost" | undefined
}
```

> `reason: "connection-lost"` устанавливается сервером, когда истёк `RECONNECT_GRACE` (20с) после дисконнекта участника — то есть звонок прекратился из-за потери сети, а не по воле пользователя.

---

#### `call:ice-candidate` — ICE-кандидат от удалённого

**Получает:** Любой участник

```json
{
  "callId":    "uuid-v4",
  "from":      "userId-remote",
  "candidate": {
    "candidate":     "candidate:...",
    "sdpMid":        "0",
    "sdpMLineIndex": 0
  }
}
```

---

#### `call:renegotiate` — Входящие переговоры медиа

**Получает:** Callee

```json
{
  "callId": "uuid-v4",
  "from":   "userId-caller",
  "offer":  "<RTCSessionDescriptionInit>"
}
```

---

#### `call:renegotiate-answer` — Ответ на переговоры

**Получает:** Caller

```json
{
  "callId": "uuid-v4",
  "from":   "userId-callee",
  "answer": "<RTCSessionDescriptionInit>"
}
```

---

#### `call:session` — Состояние сессии (ответ на `call:session:get`)

**Получает:** Запрашивающий участник

```json
{
  "callId":          "uuid-v4",
  "remoteUserId":    "userId-remote",
  "remoteUserName":  "Bob",
  "remoteUserAvatar": "emoji-or-url",
  "callType":        "audio" | "video",
  "participantRole": "caller" | "callee",
  "phase":           "incoming" | "outgoing" | "active",
  "offer":           "<SDP string>",
  "answer":          "<SDP string>",
  "startedAtMs":     1744660800000
}
```

> Если активной сессии нет, возвращается `{}` (пустой объект). Клиент обязан обработать оба варианта.

**Поле `phase`:**
| Значение | Смысл |
|----------|-------|
| `"incoming"` | Сессия существует, звонок не отвечен, текущий участник — callee |
| `"outgoing"` | Сессия существует, звонок не отвечен, текущий участник — caller |
| `"active"` | Звонок принят (`answeredAt` заполнен) |

---

#### `call:error` — Сигнальная ошибка

**Получает:** Caller (в большинстве случаев)

```json
{
  "message": "Call was not answered",
  "callId":  "uuid-v4"
}
```

Причины: истёк CALL_TIMEOUT (30с без ответа); оба участника уже в звонке.

---

## 5. Протокол восстановления сессии

### 5.1 Сценарий: рестарт приложения во время активного звонка

```
Android app crash / kill
        │
        ▼
App перезапускается
        │
        ▼
PersistedCallStore.load() → снапшот сессии (callId, role, phase)
        │
        ▼
Переход в состояние RESTORING
        │
        ▼
Socket.connect() → emit 'call:session:get'
        │
        ▼ 
Ответ 'call:session'
        ├─ phase: "active"  → needsSessionRestore = true
        │                      beginMediaRecovery() → rebuild PeerConnection
        │                      → CONNECTING → ACTIVE
        │
        └─ {} (пусто)       → сессия устарела
                               PersistedCallStore.clear()
                               → IDLE
```

### 5.2 Сценарий: кратковременная потеря сети (socket disconnect < 20с)

```
Socket disconnect
        │
        ▼
Клиент: НЕ разрушать PeerConnection
        │
        ▼
Сервер: запускает RECONNECT_GRACE таймер (20с)
        │
Socket reconnect (< 20с)
        │
        ├─ emit 'call:session:get'
        │
        ├─ phase: "active" → needsSessionRestore = false (сессия жива)
        │                    Продолжаем звонок с существующим PC
        │
        └─ Сервер: отменяет RECONNECT_GRACE таймер
```

### 5.3 Сценарий: длительная потеря сети (socket disconnect > 20с)

```
Socket disconnect
        │
        ▼
Сервер: RECONNECT_GRACE (20с) истёк
        │
        ▼
Сервер: emit 'call:ended' { reason: "connection-lost" } → удалённому участнику
        Сессия удалена из памяти
        │
        ▼
Клиент переподключается:
        emit 'call:session:get' → ответ {}
        → IDLE
```

### 5.4 Флаг `needsSessionRestore` (Android)

Критически важный флаг. Устанавливается в `true` **только** при восстановлении из `PersistedCallStore` после рестарта приложения. Сбрасывается в `false` при получении первого успешного `call:session`.

**Зачем:** Предотвращает уничтожение живой PeerConnection при обычных socket-reconnect событиях. Без него каждый реконнект сокета (сеть моргнула на 2 секунды) уничтожал бы активный звонок.

---

## 6. Константы таймаутов

### 6.1 Серверные

| Константа | Значение | Назначение |
|-----------|----------|------------|
| `DIRECT_CALL_NO_ANSWER_TIMEOUT_MS` | **30 000 мс** | Caller ждёт ответа. По истечении → `call:error` caller, сессия удаляется |
| `DIRECT_CALL_RECONNECT_GRACE_MS` | **20 000 мс** | Grace при дисконнекте участника активного звонка. По истечении → `call:ended { reason: "connection-lost" }` |
| Group disconnect grace | **8 000 мс** | Grace для участника группового звонка при socket disconnect |

### 6.2 Клиентские (Android / Web — обязательно согласовать)

| Константа | Значение | Назначение |
|-----------|----------|------------|
| `CALL_TIMEOUT_MS` | **30 000 мс** | Клиент завершает исходящий звонок, если нет ответа |
| `CONNECTION_TIMEOUT_MS` | **20 000 мс** | WebRTC connection timeout после отправки/получения answer |
| `ICE_DISCONNECTED_GRACE_MS` | **4 000 мс** | Grace после `ICE DISCONNECTED` перед recovery |
| `ICE_EARLY_STABLE_WINDOW_MS` | **8 000 мс** | Окно после ICE connected. `ICE FAILED` в этом окне обрабатывается как DISCONNECTED (grace), не как permanent fail |
| `RECOVERY_COOLDOWN_MS` | **10 000 мс** | Минимальный интервал между попытками recovery |
| `AUTO_RESET_MS` | **3 000 мс** | Задержка перед переходом из `ENDED`/`FAILED` в `IDLE` |

---

## 7. Recovery-политика

### 7.1 Принцип anti-glare

Только **Caller** (инициатор звонка, `participantRole = "caller"`) инициирует `call:renegotiate` при recovery. Callee всегда ждёт offer и отвечает answer. Это правило **не должно нарушаться** ни на одной платформе.

### 7.2 Триггеры recovery

| Триггер | Источник | Действие |
|---------|----------|----------|
| `ICE DISCONNECTED` + grace 4с | ICE state callback | `beginMediaRecovery("ice-disconnected")` |
| `ICE FAILED` (после 8с stable window) | ICE state callback | `beginMediaRecovery("ice-failed")` |
| `CONNECTION_TIMEOUT` (20с) | Timer в состоянии CONNECTING | `beginMediaRecovery("connecting-timeout")` |
| `RESTORING timeout` | Timer при восстановлении | `beginMediaRecovery("restoring-timeout")` |

### 7.3 Алгоритм recovery

```
beginMediaRecovery(trigger)
    │
    ├─ Cooldown check: если прошло < 10с с последнего recovery → отменить
    │
    ├─ Teardown: закрыть существующий PeerConnection
    │
    ├─ Rebuild: создать новый PeerConnection с теми же ICE-серверами
    │
    ├─ Caller: создать новый offer (iceRestart: true) → emit call:renegotiate
    │
    └─ Callee: ждать call:renegotiate → ответить call:renegotiate-answer
```

### 7.4 Тип ошибки ICE FAILED в early stable window

Если ICE FAILED пришёл менее чем через 8с после первого ICE CONNECTED — это транзиентная ошибка (драйвер/firmware глюк). Обрабатывать как DISCONNECTED (grace 4с), а не как permanent fail.

---

## 8. Групповые звонки

### 8.1 Топология

Full-mesh P2P: каждый участник создаёт `RTCPeerConnection` с каждым другим участником. Участник A → B создаёт PC(A→B), участник B → A создаёт PC(B→A). Итого N*(N-1)/2 соединений.

### 8.2 Ролевая модель

При входе нового участника C в комнату с A и B:
- A и B получают `group-call:peer-joined` → каждый создаёт PC и отправляет offer → C.
- C получает `group-call:room-info` со списком участников → ждёт offer от каждого.
- C отвечает answer каждому участнику.

### 8.3 Клиент → Сервер (группа)

---

#### `group-call:join` — Вход в комнату

```json
{
  "groupId":    "chat-id",
  "userId":     "my-user-id",
  "userName":   "Alice",
  "userAvatar": "emoji-or-url"
}
```

**Серверная логика:**
1. Удаляет пользователя из всех других групповых звонков (нельзя быть в двух сразу).
2. Добавляет в `groupCallRooms[groupId]`.
3. Сохраняет в `groupCallParticipants`.
4. Возвращает новому участнику `group-call:room-info`.
5. Всем остальным рассылает `group-call:peer-joined`.

---

#### `group-call:leave` — Выход из комнаты

```json
{
  "groupId": "chat-id",
  "userId":  "my-user-id"
}
```

---

#### `group-call:offer` — Offer конкретному участнику

```json
{
  "groupId": "chat-id",
  "to":      "userId-remote",
  "from":    "userId-local",
  "offer":   "<RTCSessionDescriptionInit>"
}
```

---

#### `group-call:answer` — Answer конкретному участнику

```json
{
  "groupId": "chat-id",
  "to":      "userId-remote",
  "from":    "userId-local",
  "answer":  "<RTCSessionDescriptionInit>"
}
```

---

#### `group-call:ice-candidate` — ICE для конкретного участника

```json
{
  "groupId":   "chat-id",
  "to":        "userId-remote",
  "from":      "userId-local",
  "candidate": {
    "candidate":     "candidate:...",
    "sdpMid":        "0",
    "sdpMLineIndex": 0
  }
}
```

---

#### `group-call:peer-state` — Обновление медиа-состояния

```json
{
  "groupId":         "chat-id",
  "userId":          "my-user-id",
  "isMuted":         false,
  "isScreenSharing": true
}
```

---

### 8.4 Сервер → Клиент (группа)

---

#### `group-call:room-info` — Список участников (для нового)

```json
{
  "groupId": "chat-id",
  "participants": [
    {
      "userId":          "userId-a",
      "name":            "Bob",
      "avatar":          "emoji-or-url",
      "isMuted":         false,
      "isScreenSharing": false
    }
  ]
}
```

---

#### `group-call:peer-joined` — Новый участник (для существующих)

```json
{
  "groupId":    "chat-id",
  "userId":     "userId-new",
  "userName":   "Carol",
  "userAvatar": "emoji-or-url"
}
```

---

#### `group-call:peer-left` — Участник покинул

```json
{
  "groupId": "chat-id",
  "userId":  "userId-left"
}
```

---

#### `group-call:offer` / `group-call:answer` / `group-call:ice-candidate`

Идентичны клиентским версиям, но поле `to` заменено на данные получателя (сервер уже смаршрутировал). Поле `from` сохраняется.

---

#### `group-call:peer-state`

```json
{
  "userId":          "userId-remote",
  "isScreenSharing": true,
  "isMuted":         false
}
```

---

## 9. Push-уведомления

### 9.1 Канал уведомлений

Входящие звонки отправляются по каналу `calls_incoming` (высокий приоритет).

### 9.2 Payload push-уведомления (входящий звонок)

```json
{
  "type":        "incoming_call",
  "callId":      "uuid-v4",
  "callerId":    "userId-caller",
  "callerName":  "Alice",
  "callerAvatar": "emoji-or-url",
  "callType":    "audio" | "video",
  "title":       "Входящий звонок от Alice",
  "body":        "Видеозвонок",
  "url":         "/?incomingCallerId=<encoded-userId>"
}
```

> Поле `url` — legacy для Web deep link. Нативные клиенты должны читать структурные поля (`callerId`, `callType`), не парсить `url`.

### 9.3 APNs category

`CALL_INCOMING` — для будущей интеграции с CallKit на iOS.

---

## 10. Абстрактные интерфейсы платформ

Все платформы реализуют эти интерфейсы. Бизнес-логика (стейт-машина) зависит только от интерфейсов, не от платформенных деталей.

### 10.1 `CallSignalingClient`

```typescript
interface CallSignalingClient {
    // ── Исходящие события ──────────────────────────────────────────────

    /** Инициировать звонок. Возвращает созданный callId. */
    initiateCall(params: {
        to: string;
        offer: RTCSessionDescriptionInit;
        callType: 'audio' | 'video';
        fromName: string;
        fromAvatar?: string;
        toName?: string;
        toAvatar?: string;
    }): string; // callId

    sendAnswer(callId: string, to: string, answer: RTCSessionDescriptionInit): void;
    rejectCall(callId: string, to: string): void;
    endCall(callId: string, to: string): void;
    sendIceCandidate(callId: string, to: string, candidate: RTCIceCandidateInit): void;

    /** Только Caller вызывает при recovery или screen share. */
    sendRenegotiationOffer(callId: string, to: string, offer: RTCSessionDescriptionInit): void;
    sendRenegotiationAnswer(callId: string, to: string, answer: RTCSessionDescriptionInit): void;

    /** При реконнекте сокета — запросить текущую сессию. */
    requestActiveSession(userId: string): void;

    // ── Входящие события ───────────────────────────────────────────────

    onIncomingCall(handler: (params: {
        callId: string;
        from: string;
        fromName: string;
        fromAvatar?: string;
        offer: RTCSessionDescriptionInit;
        callType: 'audio' | 'video';
        createdAt: string;
    }) => void): void;

    onCallAnswered(handler: (callId: string, from: string, answer: RTCSessionDescriptionInit) => void): void;
    onCallRejected(handler: (callId: string, from: string) => void): void;
    onCallEnded(handler: (callId: string, from: string, reason?: 'connection-lost') => void): void;
    onIceCandidateReceived(handler: (callId: string, from: string, candidate: RTCIceCandidateInit) => void): void;
    onRenegotiationOffer(handler: (callId: string, from: string, offer: RTCSessionDescriptionInit) => void): void;
    onRenegotiationAnswer(handler: (callId: string, from: string, answer: RTCSessionDescriptionInit) => void): void;
    onSessionRestored(handler: (session: CallSessionPayload | null) => void): void;
    onSignalingError(handler: (callId: string, message: string) => void): void;
}

interface CallSessionPayload {
    callId: string;
    remoteUserId: string;
    remoteUserName: string;
    remoteUserAvatar?: string;
    callType: 'audio' | 'video';
    participantRole: 'caller' | 'callee';
    phase: 'incoming' | 'outgoing' | 'active';
    offer: string;
    answer: string;
    startedAtMs: number;
}
```

---

### 10.2 `MediaEngine`

```typescript
interface MediaEngine {
    // ── Медиа-управление ───────────────────────────────────────────────

    enableCamera(enabled: boolean): Promise<void>;
    enableMicrophone(enabled: boolean): Promise<void>;
    switchCamera(): Promise<void>;

    // ── WebRTC lifecycle ───────────────────────────────────────────────

    /** Создать offer (для исходящего звонка или renegotiation). */
    createOffer(iceRestart?: boolean): Promise<RTCSessionDescriptionInit>;

    /** Принять входящий offer, вернуть answer. */
    handleRemoteOffer(offer: RTCSessionDescriptionInit): Promise<RTCSessionDescriptionInit>;

    /** Принять answer от Callee. */
    handleRemoteAnswer(answer: RTCSessionDescriptionInit): Promise<void>;

    /** Добавить ICE-кандидат (буферизует, если remote description ещё не установлена). */
    addIceCandidate(candidate: RTCIceCandidateInit): Promise<void>;

    // ── Renegotiation ──────────────────────────────────────────────────

    /** Создать offer для renegotiation (screen share, recovery). */
    createRenegotiationOffer(iceRestart?: boolean): Promise<RTCSessionDescriptionInit>;

    /** Принять renegotiation offer, вернуть answer. */
    handleRenegotiationOffer(offer: RTCSessionDescriptionInit): Promise<RTCSessionDescriptionInit>;

    /** Применить renegotiation answer. */
    handleRenegotiationAnswer(answer: RTCSessionDescriptionInit): Promise<void>;

    // ── Recovery ───────────────────────────────────────────────────────

    /** Полностью пересоздать PeerConnection (teardown + rebuild). */
    rebuild(): Promise<void>;

    // ── Колбэки ────────────────────────────────────────────────────────

    /** Удалённый медиа-поток появился (для рендера <video> или Surface). */
    onRemoteStreamAdded(handler: (stream: MediaStream) => void): void;

    /** ICE connection state изменилось. */
    onIceConnectionStateChange(handler: (state: RTCIceConnectionState) => void): void;

    /** Новый локальный ICE-кандидат готов к отправке. */
    onIceCandidate(handler: (candidate: RTCIceCandidateInit) => void): void;

    /** Опционально: подавление шума (RNnoise/платформенный). */
    setNoiseSuppression?(enabled: boolean): Promise<void>;

    destroy(): void;
}
```

---

### 10.3 `CallState` (стейт-машина — единый enum)

```typescript
enum CallState {
    IDLE,
    INITIATING,        // Создаём offer
    RINGING_OUTGOING,  // Ждём ответа собеседника
    RINGING_INCOMING,  // Показываем экран "Входящий звонок"
    CONNECTING,        // ICE negotiation
    RESTORING,         // Восстановление после рестарта приложения
    ACTIVE,            // Разговор идёт
    ENDED,             // Завершён (нормально или отклонён)
    FAILED,            // Критическая ошибка
}
```

---

### 10.4 `CallRecoveryPolicy`

```typescript
interface CallRecoveryPolicy {
    /** Нужно ли пытаться recovery при данном триггере и контексте? */
    shouldAttemptRecovery(
        trigger: 'ice-disconnected' | 'ice-failed' | 'connecting-timeout' | 'restoring-timeout',
        context: {
            mediaConnectedAtMs: number | null; // null если ICE ещё не стабилизировался
            lastRecoveryAtMs: number | null;
            currentState: CallState;
        }
    ): boolean;

    /** Задержка перед началом recovery после триггера. */
    getGraceDelayMs(trigger: string): number;

    /** Является ли текущий участник caller'ом (и обязан ли отправлять renegotiation offer)? */
    isCallerResponsibleForRenegotiation(): boolean;

    /** Зафиксировать время последней попытки recovery. */
    recordRecoveryAttempt(): void;
}
```

---

## 11. Известные ограничения

### 11.1 Отсутствует персистентность на сервере

Если **бэкенд перезапускается** во время активного звонка — сессия теряется. Клиенты получат пустой ответ на `call:session:get` и перейдут в `IDLE`. Медиа-потоки могут ещё работать через P2P (если TURN не нужен), но signaling-сессия будет мертва.

**Текущее решение:** Клиенты отображают экран окончания звонка. Будущее улучшение: Redis-персистентность для сессий.

### 11.2 Full-mesh групповые звонки не масштабируются

При N участниках каждый поддерживает N-1 PeerConnection. При N > 5-6 — деградация качества из-за upload bandwidth. Будущее улучшение: SFU (Selective Forwarding Unit, например mediasoup или LiveKit).

### 11.3 Proximity Sensor на Android не реализован

Экран не гаснет автоматически при поднесении к уху. Требует интеграции `SensorManager` + `TYPE_PROXIMITY`.

### 11.4 TelecomManager на Android не реализован

Звонки не интегрированы с системным журналом вызовов Android, не появляются в системном UI Bluetooth-гарнитур. Требует `ConnectionService`.

### 11.5 CallKit на iOS не реализован

Будущая iOS-реализация должна интегрировать `CXProvider` / `CXCallController` для нативного экрана входящего звонка на Lock Screen.

### 11.6 Отсутствует версионирование протокола

Нет механизма согласования версии при несовместимом изменении payload. При breaking change придётся координировать одновременный деплой сервера и всех клиентов.

---

*Документ поддерживается синхронно с `server/src/signaling.ts`, `android-native/feature/calls/…/CallController.kt` и `src/context/CallContext.tsx`. При изменении любого из них — обновлять этот документ.*
