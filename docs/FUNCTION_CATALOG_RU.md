# Karas Messenger: каталог модулей и функций

Документ покрывает:

- все экспортируемые функции верхнего уровня;
- ключевые named helper-функции, которые формируют архитектуру;
- крупные компоненты и контексты;
- backend и Android-native слой.

## 1. Frontend

### 1.1. Entry points

#### `src/main.tsx`

| Сущность | Назначение |
|---|---|
| `configureNativeShell` | Настраивает Android status bar в native shell |
| `initSentry` | Инициализация Sentry до рендера |
| route split `/hello` | Переключает между install-landing и основным приложением |

#### `src/App.tsx`

Главный orchestrator клиентской части.

**Экспорт**

| Функция | Назначение |
|---|---|
| `AppWrapper` | Собирает `AuthProvider`, `CallProvider`, `GroupCallProvider` и приложение |

**Top-level helpers**

| Функция/сущность | Назначение |
|---|---|
| `MobileSafeAreaFrame` | Безопасная mobile-обертка |
| `getDeliveryState` | Определяет статус доставки сообщения |
| `normalizeClientRequestId` | Нормализует optimistic request id |
| `createClientRequestId` | Генерирует request id клиента |
| `toServerMessage` | Нормализует серверное сообщение для UI |
| `compareMessages` | Сравнивает сообщения по времени и id |
| `sortMessages` | Сортирует список сообщений |
| `mergeMessages` | Склеивает optimistic и подтвержденные сообщения |
| `preserveRenderState` | Сохраняет render key и animation flags |
| `getMessengerHistoryScreenKey` | Ключ экрана для history API |
| `isMessengerHistoryState` | Проверяет internal history state |
| `getHistoryBaseState` | Читает базовый `history.state` |
| `readMessengerHistoryScreen` | Восстанавливает экран из history |
| `buildMessengerHistoryState` | Строит history state |
| `getMessengerHistoryDepth` | Читает depth navigation |
| `buildMessengerHistoryStateWithDepth` | Строит history state с depth |
| `replaceCurrentUrlPreservingHistoryState` | Меняет URL без разрушения internal history |
| `normalizeChatData` | Нормализует чат |
| `sortChats` | Сортирует список чатов |
| `normalizeMessageData` | Нормализует сообщение |
| `logPerf` | Локальный perf helper |
| `getMessageDecryptCacheKey` | Ключ кеша для дешифровки сообщения |
| `getChatPreviewDecryptCacheKey` | Ключ кеша для дешифровки preview |
| `getMessagePreviewText` | Формирует preview-текст |
| `toChatLastMessage` | Приводит сообщение к preview модели |

**Главные orchestration handlers внутри `MessengerApp`**

| Функция | Назначение |
|---|---|
| `trackChatOpen` | Трекает источник открытия чата |
| `getPublicKeyCached` | Кеширует public keys пользователей |
| `updateLocalMessages` | Базовое обновление local message state |
| `addLocalMessage` | Добавляет optimistic message |
| `patchLocalMessage` | Патчит optimistic message |
| `removeLocalMessage` | Удаляет optimistic message |
| `findLocalMessageByClientRequestId` | Ищет optimistic message по request id |
| `upsertServerMessage` | Подтверждает сообщение серверной версией |
| `replaceChatsState` | Обновляет весь чат-лист |
| `patchChatState` | Патчит один чат |
| `clearChatUnread` | Обнуляет unread count |
| `patchChatPreviewText` | Патчит preview текста |
| `patchChatLastMessage` | Обновляет последнее сообщение чата |
| `removeTypingUser` | Убирает typing user |
| `upsertTypingUser` | Добавляет typing user |
| `scheduleTypingExpiry` | Снимает typing по таймеру |
| `handleTypingStart` | Локальное начало typing |
| `handleTypingStop` | Локальная остановка typing |
| `requestChatRead` | Отправляет read receipt |
| `buildLocalMessage` | Строит optimistic message model |
| `loadChats` | Загружает чаты, дешифрует previews, обновляет notifications |
| `getOtherUserId` | Возвращает собеседника direct chat |
| `getSharedKeyForOtherUser` | Получает shared key для E2EE |
| `decryptMessageTextWithSharedKey` | Дешифрует сообщение и кеширует результат |
| `decryptMessages` | Дешифрует пачку сообщений |
| `refreshActiveMessages` | Обновляет историю активного чата |
| `handleSendMessage` | Отправляет текст/media/sticker с optimistic UI и E2EE |
| `handleEditMessage` | Редактирует сообщение |
| `handleCreateChat` | Создает direct chat |
| `handleCreateGroup` | Создает группу |
| `handleCreateChannel` | Создает канал |
| `handleTogglePinned` | Переключает pin чата |
| `handleToggleNotifications` | Переключает mute/notifications |
| `handleDeleteChat` | Удаляет или покидает чат |
| `handleBlockChatUser` | Блокирует пользователя |
| `handleUnblockChatUser` | Разблокирует пользователя |
| `handleSendMedia` | Отправляет медиа |
| `handleForwardIncomingShare` | Отправляет Android shared payload в чат |
| `handleOpenFavorites` | Открывает favorites |
| `handleOpenCallHistory` | Открывает историю звонков |
| `handleToggleReaction` | Переключает реакцию |
| `handleOpenContacts` | Переключает на контакты |
| `handleChatSelect` | Выбирает чат и синхронизирует UI/history |
| `handleJoinChannel` | Выполняет self-subscribe в канал |

### 1.2. API client

#### `src/api.ts`

| Функция | Назначение |
|---|---|
| `getServerUrl` | Возвращает backend URL |
| `getSocketServerUrl` | Возвращает Socket.IO URL |
| `apiFetch` | Базовый fetch с Bearer token |
| `apiLogin` | Логин |
| `apiGetChannelStats` | Статистика канала |
| `apiGetChannelPreview` | Preview канала |
| `apiSubscribeToChannel` | Подписка на канал |
| `apiGetChannelPreviewMessages` | Preview messages канала |
| `apiRegister` | Регистрация |
| `apiGetMe` | Текущий пользователь |
| `apiRequestPasswordReset` | Запрос reset email |
| `apiResetPassword` | Сброс пароля по token |
| `apiGetVapidPublicKey` | VAPID public key |
| `apiPushSubscribe` | Web push subscribe |
| `apiPushUnsubscribe` | Web push unsubscribe |
| `apiPushDeviceRegister` | Android device register |
| `apiPushDeviceUnregister` | Android device unregister |
| `apiGetChats` | Чат-лист |
| `apiGetMessages` | История сообщений |
| `apiSendMessage` | Отправка сообщения |
| `apiEditMessage` | Редактирование сообщения |
| `apiDeleteMessage` | Удаление сообщения |
| `apiCreateChat` | Создание direct chat |
| `apiCreateGroup` | Создание группы |
| `apiCreateChannel` | Создание канала |
| `apiGetGroupMembers` | Участники группы |
| `apiAddGroupMember` | Добавление участника |
| `apiRemoveGroupMember` | Удаление участника |
| `apiSetGroupAdmin` | Выдача/снятие admin |
| `apiUpdateGroup` | Обновление группы/канала |
| `apiDeleteChat` | Удаление/leave chat |
| `apiUpdateChatPreferences` | Обновление pinned/notifications |
| `apiBlockUser` | Блокировка пользователя |
| `apiUnblockUser` | Разблокировка пользователя |
| `apiSearchUsers` | Поиск пользователей |
| `apiGetPublicKey` | Public key пользователя |
| `apiUpdatePublicKey` | Обновление собственного public key |
| `apiUploadFile` | Upload файла |
| `apiUpdateProfile` | Обновление профиля |
| `apiChangePassword` | Смена пароля и key backup |
| `apiToggleReaction` | Toggle reaction |
| `apiGetDisabledEmojis` | Чтение disabled emojis |
| `apiSetDisabledEmojis` | Запись disabled emojis |

### 1.3. E2EE

#### `src/crypto.ts`

| Функция | Назначение |
|---|---|
| `generateKeyPair` | Генерирует ECDH key pair |
| `exportPublicKey` | Экспорт public key в JWK |
| `exportPrivateKey` | Экспорт private key в JWK |
| `importPublicKey` | Импорт public key |
| `importPrivateKey` | Импорт private key |
| `deriveSharedKey` | Derive shared AES key |
| `encryptMessage` | Шифрует сообщение |
| `decryptMessage` | Дешифрует сообщение |
| `deriveKeyFromPassword` | Строит backup key из пароля |
| `encryptPrivateKey` | Шифрует private key паролем |
| `decryptPrivateKey` | Восстанавливает private key |
| `savePrivateKeyLocal` | Сохраняет private key локально |
| `loadPrivateKeyLocal` | Загружает private key локально |
| `hasPrivateKeyLocal` | Проверяет наличие local key |
| `getOrDeriveSharedKey` | Возвращает shared key из кеша или derive'ит заново |

### 1.4. Notifications

#### `src/notifications.ts`

| Функция | Назначение |
|---|---|
| `isIncomingCallPushPayload` | Типизация payload входящего звонка |
| `isNativePushSupported` | Проверка Android native режима |
| `areBrowserNotificationsSupported` | Проверка browser notifications |
| `areBrowserNotificationsAllowedInThisContext` | Проверка secure context |
| `isPushSupported` | Проверка web push API |
| `getNotificationsEnabled` | Читает глобальную настройку уведомлений |
| `hasNotificationsPreference` | Проверяет наличие сохраненной настройки |
| `setNotificationsEnabled` | Сохраняет настройку уведомлений |
| `getBrowserNotificationsEnabled` | Совместимый getter |
| `setBrowserNotificationsEnabled` | Совместимый setter |
| `initializeNotificationsEnabledFromPermission` | Инициализирует preference из permission |
| `getBrowserNotificationPermission` | Browser permission state |
| `requestBrowserNotificationPermission` | Запрос browser permission |
| `getNativeNotificationPermission` | Native permission state |
| `requestNativeNotificationPermission` | Запрос native permission |
| `getNotificationPermissionState` | Унифицированный permission getter |
| `getPendingChatOpenId` | pending chat open id |
| `consumePendingChatOpenId` | Читает и очищает pending chat open |
| `consumePendingIncomingCallerId` | Читает и очищает pending caller |
| `createAndroidNotificationChannels` | Создает Android channels |
| `clearNativeChatNotifications` | Чистит native chat notifications |
| `showChatNotification` | Показывает локальное notification |
| `registerPushServiceWorker` | Регистрирует SW |
| `getPushSubscription` | Возвращает текущую push subscription |
| `subscribeToPush` | Оформляет push subscribe |
| `unsubscribeFromPush` | Выполняет push unsubscribe |
| `ensurePushSubscription` | Поддерживает web push subscription |
| `ensureNativePushRegistration` | Поддерживает Android push registration |
| `unregisterNativePush` | Снимает Android push registration |

### 1.5. Contexts

#### `src/context/AuthContext.tsx`

| Функция | Назначение |
|---|---|
| `AuthProvider` | Держит auth state и auth actions |
| `useAuth` | Доступ к auth context |

#### `src/context/ThemeContext.tsx`

| Функция | Назначение |
|---|---|
| `ThemeProvider` | Управляет темой и синхронизирует ее с Android status bar |
| `useTheme` | Доступ к theme context |

#### `src/context/CallContext.tsx`

| Экспорт | Назначение |
|---|---|
| `useCall` | Доступ к direct-call context |
| `CallProvider` | Полный lifecycle индивидуального звонка |

**Ключевые helper-функции**

- `hasAudioOutputSelectionSupport`
- `createMixedAudioTrack`
- `inferFacingModeFromLabel`
- `getPreferredVideoConstraints`
- `openPreferredVideoStream`
- `delay`
- `canSwitchCameraSources`
- `fetchIceServers`
- `createIdleCallState`

**Главные действия provider**

- `initiateCall`
- `acceptCall`
- `rejectCall`
- `endCall`
- `toggleMute`
- `toggleNoiseSuppression`
- `toggleVideo`
- `switchCamera`
- `startScreenShare`
- `stopScreenShare`
- `selectAudioInput`
- `selectVideoInput`
- `selectAudioOutput`

#### `src/context/GroupCallContext.tsx`

| Экспорт | Назначение |
|---|---|
| `useGroupCall` | Доступ к group-call context |
| `GroupCallProvider` | Полный lifecycle группового звонка |

**Ключевые helper-функции**

- `createMixedAudioTrack`
- `getPreferredVideoConstraints`
- `fetchIceServers`
- `createIdleGroupCallState`
- `hasOwn`

**Главные действия**

- `joinGroupCall`
- `leaveGroupCall`
- `toggleMute`
- `toggleNoiseSuppression`
- `toggleVideo`
- `startScreenShare`
- `stopScreenShare`
- `selectAudioInput`
- `selectVideoInput`
- `selectAudioOutput`

### 1.6. Sticker subsystem

#### `src/features/stickers/api.ts`

| Функция | Назначение |
|---|---|
| `apiGetStickerPacks` | Список packs |
| `apiGetStickerPackManifest` | Manifest pack'а |
| `apiGetStickerUserState` | Sticker user state |
| `apiSaveStickerUserState` | Сохранение user state |
| `apiCreateCustomSticker` | Создание custom sticker |
| `apiImportTelegramStickerPack` | Импорт Telegram pack |
| `apiUpdateStickerEmoji` | Обновление emoji sticker |
| `apiDeleteSticker` | Удаление sticker |

#### `src/features/stickers/repository.ts`

| Функция | Назначение |
|---|---|
| `loadStickerCatalogState` | Загружает catalog/manifests |
| `loadStickerUserState` | Загружает user state |
| `installStickerPack` | Устанавливает pack |
| `removeStickerPack` | Удаляет pack |
| `createCustomStickerPackItems` | Создает custom stickers из файлов |
| `importTelegramStickerPack` | Импортирует Telegram pack |
| `updateOwnedStickerEmoji` | Изменяет emoji собственного sticker |
| `deleteOwnedStickerItem` | Удаляет собственный sticker |
| `updateStickerUserState` | Записывает user state |
| `toggleFavoriteSticker` | Переключает favorite |
| `recordStickerSent` | Обновляет recent/frequent |
| `getStickerById` | Ищет sticker по id |

#### `src/features/stickers/state.ts`

| Функция | Назначение |
|---|---|
| `useStickerManager` | Главный hook управления stickers |

#### `src/features/stickers/ui/*`

| Компонент | Назначение |
|---|---|
| `LottieSticker` | Рендер animated sticker |
| `StickerPanel` | UI выбора стикеров |
| `StickerRenderer` | Унифицированный sticker renderer |
| `StickerVideo` | Рендер video sticker |

### 1.7. Hooks

#### `src/hooks/useCallHistory.ts`

| Функция | Назначение |
|---|---|
| `emitCallRecord` | Публикует запись звонка в глобальный event bus |
| `useCallHistory` | Ведет историю звонков пользователя |

### 1.8. Основные компоненты

| Файл | Компонент | Назначение |
|---|---|---|
| `src/pages/AuthPages.tsx` | `AuthPages` | Логин, регистрация, forgot/reset password |
| `src/HelloLanding.tsx` | `HelloLanding` | Install landing |
| `src/components/Sidebar.tsx` | `Sidebar` | Боковая навигация |
| `src/components/ChatList.tsx` | `ChatList` | Список чатов, фильтры, создание чатов/групп/каналов |
| `src/components/ChatArea.tsx` | `ChatArea` | Лента сообщений, поиск, context menu, reply, forward, reactions |
| `src/components/MessageInput.tsx` | `MessageInput` | Текст, emoji, voice, files, round video |
| `src/components/ChatInfoPanel.tsx` | `ChatInfoPanel` | Информация о чате, канале, группе, настройках |
| `src/components/SettingsModal.tsx` | `SettingsModal` | Настройки профиля, пароля, E2EE и приложения |
| `src/components/CallModal.tsx` | `CallModal` | Модал входящего/исходящего звонка |
| `src/components/ActiveCall.tsx` | `ActiveCall` | Экран активного звонка |
| `src/components/ActiveGroupCall.tsx` | `ActiveGroupCall` | Экран группового звонка |
| `src/components/CallHistoryModal.tsx` | `CallHistoryModal` | История звонков |
| `src/components/ForwardModal.tsx` | `ForwardModal` | Выбор чата для пересылки |
| `src/components/CreateGroupModal.tsx` | `CreateGroupModal` | Создание группы или канала |
| `src/components/GroupMembersModal.tsx` | `GroupMembersModal` | Управление участниками |
| `src/components/ChatActionsModal.tsx` | `ChatActionsModal` | Быстрые действия над чатом |
| `src/components/MessageContextMenu.tsx` | `MessageContextMenu` | Действия над сообщением |
| `src/components/DateNavigatorModal.tsx` | `DateNavigatorModal` | Переход по датам истории |
| `src/components/MediaViewer.tsx` | `MediaViewer` | Полноэкранный просмотр медиа |
| `src/components/AudioPlayer.tsx` | `AudioPlayer` | Воспроизведение аудио |
| `src/components/RoundVideoPlayer.tsx` | `RoundVideoPlayer` | Рендер round video |
| `src/components/NotificationBootstrapper.tsx` | `NotificationBootstrapper` | Bootstrap notifications/push |
| `src/components/MobileNotificationsBanner.tsx` | `MobileNotificationsBanner` | Баннер включения уведомлений |
| `src/components/UserMenu.tsx` | `UserMenu` | Меню пользователя |

### 1.9. Utilities

| Файл | Функции | Назначение |
|---|---|---|
| `src/sentry.ts` | `initSentry`, `addSentryBreadcrumb`, `captureException`, `setSentryUser` | Observability |
| `src/utils/androidIncomingShare.ts` | `fileFromAndroidIncomingShare`, `summarizeAndroidIncomingShare` | Android incoming share |
| `src/utils/avatarPreview.ts` | `isImageAvatar`, `getAvatarPreviewUrl` | Preview аватаров |
| `src/utils/callAvatar.ts` | `resolveCallAvatar` | Унифицированный avatar для звонков |
| `src/utils/callDevices.ts` | `hasBrowserAudioOutputSelectionSupport`, `hasScreenShareSupport`, `getDisplayMediaStream`, `getAudioProcessingConstraints`, `getAudioConstraints`, `getVideoConstraints`, `enumerateInputDevices`, `enumerateAudioOutputs`, `applyAudioOutputSelection` | Media devices и audio routing |
| `src/utils/callSounds.ts` | `startRingtone`, `stopRingtone`, `startRingback`, `stopRingback`, `playConnectedSound`, `playEndCallSound`, `playRejectedSound`, `stopAllCallSounds` | Call sounds |
| `src/utils/messageText.ts` | `tokenizeMessageText`, `extractMessageLinks` | Парсинг текста и ссылок |
| `src/utils/nativeAndroidScreenShare.ts` | `hasNativeAndroidScreenShareSupport`, `createNativeAndroidScreenShareSession` | Native Android screen share bridge |
| `src/utils/presence.ts` | `formatLastSeen`, `getChatPresenceStatus` | Presence статус |
| `src/utils/rnnoise.ts` | `canUseRnnoiseWorklet`, `createRnnoiseProcessedTrack` | Noise suppression |
| `src/utils/runtimeContext.ts` | `isNativeAndroidShell`, `isStandalonePwa`, `isLikelyMobileDevice`, `shouldSkipChatOpenAutoFocus` | Runtime detection |
| `src/utils/systemBack.ts` | `registerSystemBackHandler`, `dispatchSystemBack`, `useSystemBackHandler` | Системная кнопка назад |
| `src/utils/textEncoding.ts` | `decodeMaybeMojibake`, `normalizeMaybeMojibake` | Исправление mojibake |

### 1.10. Plugin contract

#### `src/plugins/androidCallBridge.ts`

| Метод | Назначение |
|---|---|
| `getAudioDevices` | Список audio devices |
| `selectAudioOutput` | Переключение output |
| `startScreenCapture` | Запуск native screen capture |
| `stopScreenCapture` | Остановка native screen capture |
| `getRuntimeCapabilities` | Runtime capabilities shell |
| `getNativePushConfig` | Push-конфиг |
| `consumePendingIncomingShare` | Получение pending share payload |
| `shareAppApk` | Открытие share-sheet для APK |

## 2. Backend

### 2.1. Entry point

#### `server/src/index.ts`

| Сущность | Назначение |
|---|---|
| `main` | Инициализирует БД, push, Express, API, static handlers и signaling |

#### `server/src/bootstrap.ts`

| Сущность | Назначение |
|---|---|
| `dotenv/config` | Поднимает env |
| `./index.js` | Запускает сервер |

### 2.2. Database

#### `server/src/db.ts`

| Функция | Назначение |
|---|---|
| `initDB` | Инициализация БД и миграции |
| `getDB` | Возвращает singleton БД |
| `getAppSetting` | Читает `app_settings` |
| `setAppSetting` | Пишет `app_settings` |
| `saveDB` | Сохраняет sql.js БД на диск |

### 2.3. Auth и email

#### `server/src/auth.ts`

| Функция | Назначение |
|---|---|
| `authMiddleware` | Проверяет JWT и прикрепляет `req.userId` |

**Helpers**

- `buildAppUrl`
- `hashResetToken`
- `cleanupExpiredPasswordResetTokens`

**Роуты**

- `POST /register`
- `POST /login`
- `POST /change-password`
- `POST /request-password-reset`
- `POST /reset-password`

#### `server/src/email.ts`

| Функция | Назначение |
|---|---|
| `normalizeEmail` | Нормализация email |
| `isValidEmail` | Валидация email |

#### `server/src/mailer.ts`

| Функция | Назначение |
|---|---|
| `isMailConfigured` | Проверка SMTP-конфига |
| `sendPasswordResetEmail` | Отправка reset email |

### 2.4. REST API

#### `server/src/routes.ts`

**Ключевые helper-функции**

| Функция | Назначение |
|---|---|
| `resolveUploadedMediaType` | Определяет media type upload-файла |
| `normalizeUsernameInput` | Нормализует username |
| `buildChatPayload` | Собирает payload чата для клиента |
| `validatePushDeviceRegistration` | Валидирует Android push device payload |
| `validatePushDeviceUnregister` | Валидирует unregister payload |
| `normalizeMessageContentType` | Нормализует `content_type` |
| `buildStickerMessagePayloadFromColumns` | Собирает sticker payload из DB row |
| `buildMessageResponse` | Собирает client-ready message payload |
| `requireChatMember` | Проверка membership |
| `requireGroupMember` | Проверка membership group |
| `requireChatAdmin` | Проверка admin rights |
| `requireChatOwner` | Проверка owner rights |
| `requireGroupAdmin` | Проверка group admin |
| `requireChatIsGroup` | Проверка типа чата |
| `getChatType` | Возвращает normalized chat type |

**Основные маршруты**

- `GET /users/me`
- `GET /users/me/stickers/state`
- `PUT /users/me/stickers/state`
- `GET /stickers/packs`
- `GET /stickers/packs/:id`
- `POST /stickers/custom`
- `PATCH /stickers/:stickerId`
- `DELETE /stickers/:stickerId`
- `POST /stickers/import/telegram`
- `GET /push/vapid-public-key`
- `POST /push/subscribe`
- `POST /push/unsubscribe`
- `POST /push/devices/register`
- `POST /push/devices/unregister`
- `PUT /users/me`
- `GET /users/:id/public-key`
- `PUT /users/public-key`
- `GET /users/search`
- `GET /chats`
- `PUT /chats/:id/preferences`
- `GET /chats/:id/messages`
- `GET /channels/:id/preview/messages`
- `POST /chats/:id/messages`
- `PUT /messages/:id`
- `DELETE /messages/:id`
- `POST /chats`
- `POST /groups`
- `POST /channels`
- `PUT /groups/:id`
- `GET /groups/:id/members`
- `GET /channels/:id/stats`
- `GET /channels/:id/preview`
- `POST /channels/:id/subscribe`
- `POST /groups/:id/members`
- `DELETE /groups/:id/members/:userId`
- `PUT /groups/:id/members/:userId/admin`
- `DELETE /chats/:id`
- `POST /users/:id/block`
- `DELETE /users/:id/block`
- `POST /upload`
- `POST /messages/:id/reactions`
- `GET /groups/:id/disabled-emojis`
- `PUT /groups/:id/disabled-emojis`

### 2.5. Signaling

#### `server/src/signaling.ts`

| Функция | Назначение |
|---|---|
| `makeGroupCallDisconnectKey` | Ключ pending disconnect |
| `clearPendingGroupCallDisconnect` | Снимает pending disconnect timeout |
| `bindSocketToUser` | Привязывает userId к socket |
| `emitIncomingCall` | Транслирует входящий direct call |
| `clearPendingCall` | Удаляет pending call |
| `schedulePendingCallTimeout` | Timeout для неотвеченного звонка |
| `initializeSignaling` | Поднимает Socket.IO event layer |

### 2.6. Push

#### `server/src/push.ts`

| Функция | Назначение |
|---|---|
| `initPush` | Инициализирует VAPID и Firebase |
| `getVapidPublicKey` | Возвращает VAPID public key |
| `validateSubscriptionShape` | Валидирует browser push subscription |
| `upsertSubscriptionForUser` | Сохраняет web subscription |
| `removeSubscriptionForUser` | Удаляет web subscription |
| `listSubscriptionsForUser` | Список web subscriptions |
| `registerPushDevice` | Регистрирует Android device |
| `unregisterPushDevice` | Удаляет Android device |
| `listPushDevicesForUser` | Список Android devices |
| `sendFcmToUser` | Отправляет FCM push |
| `sendPushToUser` | Отправляет push и в web, и в Android |

### 2.7. Read receipts

#### `server/src/readReceipts.ts`

| Функция | Назначение |
|---|---|
| `markChatMessagesRead` | Отмечает непрочитанные сообщения как прочитанные |
| `emitChatMessagesRead` | Отправляет read receipts участникам |

### 2.8. Stickers

#### `server/src/stickers.ts`

| Функция | Назначение |
|---|---|
| `getDefaultStickerUserState` | Пустой sticker user state |
| `normalizeStickerUserState` | Нормализация sticker state |
| `getStickerUserState` | Получение sticker state пользователя |
| `saveStickerUserState` | Сохранение state |
| `upsertStickerUsageState` | Обновление usage/recent |
| `getFrequentStickerIds` | Вычисление frequent stickers |
| `getStickerPackSummaries` | Summary packs |
| `getStickerPackManifest` | Полный manifest |
| `getStickerPayload` | Payload sticker для сообщения |
| `createCustomSticker` | Создание собственного sticker |
| `updateOwnedStickerEmoji` | Обновление emoji sticker |
| `deleteOwnedSticker` | Удаление sticker |
| `seedStickerCatalog` | Засев стартового sticker catalog |

### 2.9. System channel

#### `server/src/systemChannel.ts`

| Функция | Назначение |
|---|---|
| `isSystemUpdatesChannel` | Проверяет системный канал |
| `ensureSystemUpdatesChannel` | Создает и поддерживает системный канал |
| `subscribeUserToSystemUpdatesChannel` | Подписывает пользователя |
| `leaveSystemUpdatesChannel` | Opt-out из системного канала |
| `createSystemUpdatesMessage` | Создает системное сообщение |
| `promoteSystemUpdatesChannelAdminByUsername` | Назначает admin'а канала |

### 2.10. Telegram import

#### `server/src/telegramStickerImport.ts`

| Функция | Назначение |
|---|---|
| `importTelegramStickerPack` | Импорт sticker pack из Telegram |

## 3. Android native

### 3.1. Main activity

#### `android/app/src/main/java/app/karas/messenger/MainActivity.java`

| Метод | Назначение |
|---|---|
| `onCreate` | Регистрация plugin, обработка incoming share, очистка уведомлений, запрос permissions |
| `onNewIntent` | Обработка новых intents |
| `maybeRequestInitialMediaPermissions` | Первичный запрос camera/audio/media permissions |
| `addPermissionIfMissing` | Проверка и добавление permission |
| `clearNotificationsForIntent` | Очистка уведомлений открытого чата |
| `shouldCancelNotification` | Логика выбора уведомлений на отмену |
| `cancelNotification` | Отмена конкретного уведомления |
| `isChatNotification` | Проверка принадлежности notification чату |
| `extractChatIdFromIntent` | Извлечение chatId из intent |
| `extractChatIdFromBundle` | Извлечение chatId из extras |
| `extractChatIdFromUrl` | Извлечение `openChat` из URL |

### 3.2. AndroidCallBridge plugin

#### `android/app/src/main/java/app/karas/messenger/AndroidCallBridgePlugin.java`

| Метод | Назначение |
|---|---|
| `getAudioDevices` | Список audio devices |
| `selectAudioOutput` | Переключение audio output |
| `startScreenCapture` | Запуск native screen capture |
| `stopScreenCapture` | Остановка screen capture |
| `getRuntimeCapabilities` | Runtime capabilities |
| `getNativePushConfig` | Native push config |
| `consumePendingIncomingShare` | Возврат pending share payload |
| `shareAppApk` | APK share-sheet |
| `captureIncomingShare` | Захват incoming share |
| `handleStartScreenCaptureResult` | Обработка разрешения screen capture |
| `onScreenCaptureFrame` | Передача кадра в JS |
| `onScreenCaptureStopped` | Передача stop события в JS |
| `buildAudioOutputs` | Формирует outputs |
| `buildAudioInputs` | Формирует inputs |
| `applyAudioOutput` | Применяет output device |
| `prepareForCommunication` | Подготовка AudioManager |
| `isFirebaseConfigured` | Проверка Firebase |
| `getSelectedAudioOutputId` | Текущий output |
| `hasLegacyEarpiece` | Проверка earpiece |
| `copyShareableApkToCache` | Копирование APK в cache |
| `getAppNameForShare` | App name для share |
| `toDeviceOption` | Преобразование устройства в JS объект |
| `getDeviceId` | Формирует device id |
| `isOutputDevice` | Проверка output device |
| `isInputDevice` | Проверка input device |
| `getDeviceLabel` | Название устройства |

### 3.3. Screen capture service

#### `android/app/src/main/java/app/karas/messenger/AndroidCallBridgeMediaProjectionService.java`

| Метод | Назначение |
|---|---|
| `setListener` | Устанавливает listener plugin-слоя |
| `resolveCaptureConfig` | Расчет параметров захвата экрана |
| `onCreate` | Создание notification channel |
| `onStartCommand` | Запуск или остановка foreground service |
| `onDestroy` | Корректное завершение session |
| `onBind` | Service не использует bind |
| `startCaptureSession` | Инициализация MediaProjection, ImageReader, VirtualDisplay |
| `handleImageAvailable` | Кодирует кадр в JPEG base64 |
| `stopCaptureSession` | Остановка capture session |
| `dispatchStoppedOnce` | Единичная отправка stop события |
| `buildNotification` | Строит foreground notification |
| `createNotificationChannel` | Создает notification channel |
| `stopForegroundCompat` | Совместимое завершение foreground mode |
| `getParcelableIntentExtra` | Совместимое чтение parcelable extra |

### 3.4. Share and URL parsing

#### `AndroidIncomingShareParser.java`

| Метод | Назначение |
|---|---|
| `parse` | Парсит Android share intent |
| `extractSharedText` | Извлекает текст |
| `extractSharedUris` | Извлекает URI файлов |
| `importSharedUri` | Копирует shared URI в cache и описывает файл |
| `resolveDisplayName` | Определяет display name |
| `resolveExtension` | Подбирает расширение |

#### `OpenChatTargetParser.java`

| Метод | Назначение |
|---|---|
| `extractOpenChatId` | Извлекает `openChat` из URL |
| `extractFromUri` | Парсит URI |
| `extractQueryParam` | Извлекает query param |
| `decode` | URL decode |

#### `ScreenCaptureConfigMath.java`

| Метод | Назначение |
|---|---|
| `clampRequestedMax` | Ограничение requested max |
| `resolveScale` | Расчет scale |
| `resolveTargetDimension` | Вычисление целевой dimension |
| `resolveFrameRate` | Ограничение frame rate |
| `resolveJpegQuality` | Ограничение JPEG quality |
| `clamp` | Универсальный clamp |
| `alignDimension` | Выравнивание размера до четного |

## 4. PWA и public

### `public/sw.js`

| Сценарий | Назначение |
|---|---|
| `install` | Мгновенная активация SW |
| `activate` | Захват клиентов |
| `safeJson` | Безопасный разбор payload |
| `push` handler | Показ notification или прокидка payload в открытое окно |
| `notificationclick` handler | Фокус/навигация по notification |

### `public/manifest.webmanifest`

Определяет:

- standalone display mode;
- icon set;
- theme colors;
- push ecosystem metadata.

## 5. Итог

Текущая кодовая база уже содержит:

- зрелый client orchestration слой;
- полноценный backend;
- real-time коммуникации;
- E2EE для direct chats;
- sticker subsystem;
- Android-native extension layer.

Это сильный фундамент и для поддержки продукта, и для его дальнейшей коммерциализации.
