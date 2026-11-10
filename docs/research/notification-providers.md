# Хостед-провайдеры in-app-нотификаций для Cadence

Вопрос: [#2](https://github.com/kostachernyshov/cohort-005-project/issues/2). Какой хостед-провайдер возьмёт на себя in-app-ленту Cadence (колокольчик, лента, read/unread), чтобы не заводить свою таблицу нотификаций в SQLite. Cadence: React Router 7 (framework mode), server-side actions, Drizzle + SQLite.

Сравнивали Knock, Novu, MagicBell, Courier и SuprSend. Источники: официальные страницы с ценами и документация, проверено 2026-11-11. Если документация о чём-то молчит, в таблице стоит «не указано», а не догадка.

## Итог

**Финалисты: Knock и Novu.**

- **Knock** — лучший вариант, если self-host не нужен. В бесплатном тарифе 10 000 сообщений в месяц, неограниченное число окружений, мест и профилей пользователей; лента и Feed API входят в Free ([pricing](https://knock.app/pricing)). Есть и готовые React-компоненты (`NotificationFeedPopover`, `NotificationIconButton`), и headless-хуки (`useNotifications`, `useNotificationStore`) в одном пакете `@knocklabs/react`. Real-time идёт через управляемый websocket. Клиент авторизуется подписанным на бэкенде JWT (RS256), подписать его можно прямо в server loader React Router, без лишнего сетевого запроса. Минусы: следующий тариф стоит сразу $250/мес, а на Free остаётся брендинг Knock.
- **Novu** — лучший вариант, если важны дешёвый следующий шаг и путь отхода. Free: 10 000 workflow runs в месяц, окружения Dev и Prod, неограниченное число подписчиков. Pro стоит от $30/мес ([pricing](https://novu.co/pricing)). Ядро под MIT, есть community self-hosted edition ([GitHub](https://github.com/novuhq/novu), [self-hosting](https://docs.novu.co/community/self-hosting-novu/overview)), так что при необходимости можно уйти с облака. Есть `<Inbox />` и headless-хуки (`useNotifications`, `useCounts`) в `@novu/react`. Авторизация клиента: HMAC-SHA256 от `subscriberId`. Минусы: на Free всего 5 workflows и 24 часа хранения activity feed (это лог для отладки, а не сама лента); self-host требует MongoDB, два Redis и S3.

Остальные отсеяны. У **MagicBell** самый маленький Free (1 000 доставок), а следующий тариф стоит $249. У **Courier** щедрый Free (10 000 отправок), но headless-слой документирован слабее, а self-host нет. У **SuprSend** хороший Free и самый дешёвый платный тариф ($100), но брендинг на ленте во Free, а self-host только в Enterprise. Courier и SuprSend остаются запасными вариантами.

## Сводная таблица

| | Knock | Novu | MagicBell | Courier | SuprSend |
|---|---|---|---|---|---|
| Free | 10 000 сообщений/мес, окружения ∞, места ∞, брендинг Knock [1] | 10 000 workflow runs/мес, 5 workflows, 2 окружения, 3 места [2] | 1 000 доставок/мес, 1 проект [3] | 10 000 отправок/мес [4] | 10 000 нотификаций/мес, брендинг SuprSend на ленте [5] |
| Следующий тариф | Starter $250/мес: 50k, потом $0.005 за сообщение [1] | Pro от $30/мес: 30k, потом $1.20 за 1k [2] | Startup $249/мес: 50k, потом $0.0025 [3] | Business: $0.005 за отправку, pay-as-you-go [4] | Essentials $100/мес: 50k, потом $2 за 1k [5] |
| Готовая React-лента | `NotificationFeedPopover`, `NotificationFeed`, `NotificationIconButton` [6] | `<Inbox />` (Bell, Notifications, Preferences) [8] | `@magicbell/react`: `Inbox`, `FloatingInbox` [11] | `@trycourier/courier-react`: `CourierInbox` [13] | `@suprsend/react` (drop-in) [15] |
| Headless | `useNotifications`, `useNotificationStore`, `feedClient.markAsSeen` и др. [7] | `useNotifications`, `useCounts`, `useNovu` [9] | В новом `@magicbell/react` хуки не документированы; в старом `@magicbell/react-headless` есть `useNotifications`, `useBell` [11][12] | `useCourier().inbox`: `feeds`, `totalUnreadCount`, `load()`, `listenForUpdates()` [14] | `@suprsend/react-core`: `useFeedData`, `useFeedClient`, `markAsRead` [16] |
| Real-time | Управляемый websocket [17] | Есть (`useCounts` и `useNotifications` обновляются сами), транспорт websocket (`socketUrl wss://…`) [9][10] | Есть, через `MagicBellProvider` (по старому SDK) [12] | WebSocket через `listenForUpdates()` [14] | «feed socket» [16] |
| Пользователи | `id` = ваш ключ; upsert через identify или inline при trigger; пользователи свои в каждом окружении [18] | Создаются just-in-time из Inbox или через API [10] | `user_external_id` и/или `user_email` в JWT [19] | Регистрируются при `signIn`, заранее создавать не нужно [20] | `entity_id` = ваш ID [21] |
| Авторизация клиента | JWT RS256 с ключом подписи, `sub` = user id, по умолчанию 1 ч, `onUserTokenExpiring` [6b] | HMAC-SHA256 (`subscriberId`, secret key) → `subscriberHash` [10] | JWT HS256 с секретом проекта [19] | JWT от `POST /auth/issue-token` (scopes, `expires_in`), SDK сам не обновляет [20] | JWT ES256, `refreshUserToken` [21] |
| Тестовые окружения | Окружения ∞ на всех тарифах (Development/Production) [1][18] | Dev + Prod на Free, до 10 на Team [2] | Проекты (1 на Free) [3] | API-ключи свои у каждого окружения [20] | Workspaces Sandbox, Staging, Production [22] |
| Self-host | Нет | Да: ядро MIT, community edition; `/enterprise` под коммерческой лицензией [23][24] | Нет | Нет | Только Enterprise [5] |

## Детали

### Knock

- **Цены.** Free: $0, до 10 000 сообщений в месяц, окружения и места без ограничений, профили пользователей без ограничений. Лента и Feed API входят во Free, убрать брендинг Knock во Free нельзя. Starter: $250/мес плюс usage, включено 50 000, дальше $0.005 за сообщение [1].
- **UI.** В `@knocklabs/react` есть готовые `NotificationFeedPopover`, `NotificationFeed` и `NotificationIconButton` (бейдж unread/unseen) и полный набор хуков [6]. Headless: `useKnockClient`, `useNotifications(client, channelId)`, `useNotificationStore` (items, metadata), а также `markAsSeen`/`markAllAsSeen` [7].
- **Real-time.** Knock держит управляемый websocket и пушит новые сообщения в in-app-канал [17].
- **Авторизация (enhanced security mode).** Бэкенд подписывает JWT ключом RS256 из дашборда (`KNOCK_SIGNING_KEY`). `sub` = user id, `exp`/`iat` рекомендуются, `signUserToken` по умолчанию даёт 3600 с. Клиенту токен передаётся как `userToken` в `KnockProvider`, перед истечением срабатывает `onUserTokenExpiring` [6b].
- **Пользователи.** Пользователя нужно синхронизировать (identify) до отправки. `id` = ваш первичный ключ (до 256 символов, без `/` и `#`). Обновлять можно через identify или inline в trigger workflow. Пользователи изолированы по окружениям [18].
- **Что это значит для Cadence.** Токен подписывается в `loader` корневого роута по id сессионного пользователя. Из server actions (например, при одобрении refund request) вызывается `workflows.trigger` из Node SDK с inline identify, так что отдельная синхронизация пользователей не нужна.

### Novu

- **Цены.** Free: 10 000 workflow runs в месяц, 5 workflows, 2 окружения (Dev и Prod), подписчиков без ограничений, 60 RPS, 24 ч хранения activity feed, 3 места, кредитная карта не нужна. Pro: от $30/мес, 30 000+ runs, дальше $1.20 за 1 000, 20 workflows, 7 дней хранения. Team: от $250/мес [2].
- **UI.** `@novu/react` (React 18+, поддерживается 19): `<Inbox />` с подкомпонентами Bell, Notifications, InboxContent, Preferences [8]. Headless: `useNotifications` (notifications, hasMore, fetchMore, действия read/archive), `useCounts` (unread, unseen), `useNovu().on()` для своих событий. Хуки сами слушают real-time-события [9].
- **Авторизация.** Включаете HMAC в настройках In-App-провайдера для production-окружения. На сервере считаете `HMAC-SHA256(subscriberId, secretKey)` в hex и передаёте его в проп `subscriberHash`. Если HMAC включён, а хеша нет, Inbox не загрузится [10].
- **Пользователи.** Подписчик создаётся just-in-time из Inbox по объекту `subscriber`. Чтобы обновить существующий профиль, нужен `subscriberHash` [10].
- **Self-host.** Ядро под MIT, каталог `/enterprise` под коммерческой лицензией [24]. В документации есть «Community self-hosted edition»: сервисы API, Worker, WS, Dashboard плюс MongoDB, два Redis (у одного включён AOF) и S3-совместимое хранилище. Социального логина нет, часть облачных функций недоступна [23]. На странице цен self-host с поддержкой указан как опция Enterprise [2].
- **Оговорка.** 24 часа на Free — это хранение лога activity feed (наблюдаемость), а не ленты Inbox. Сколько хранятся сами сообщения Inbox на Free, в просмотренных источниках не нашли. Проверить до выбора.

### MagicBell

- **Цены.** Builder (Free): 1 000 доставок в месяц, 1 проект. Каждая доставка в каждый канал считается отдельно. Startup: $249/мес, 50 000, дальше $0.0025 за доставку, 5 проектов [3].
- **UI.** `@magicbell/react`: `ContextProvider`, `Inbox`, `FloatingInbox`, `ToastContainer` и другие. Headless-хуков на этой странице нет [11]. В более старом `@magicbell/react-headless` есть `useNotifications`, `useBell` (unreadCount в реальном времени), `useMagicBellEvent` [12].
- **Авторизация.** User JWT подписывается HS256 секретом проекта. Claims: `user_external_id` и/или `user_email` плюс `api_key` [19].
- **Вывод.** Free слишком маленький даже для разработки с реальными данными. Не берём.

### Courier

- **Цены.** Developer: бесплатно, 10 000 отправок в месяц. Business: $0.005 за отправку. Квоты для Inbox и число окружений на странице цен не указаны [4].
- **UI.** `@trycourier/courier-react` v9 (React 18+): `CourierInbox`, `CourierToast`. Старые `@trycourier/react-inbox` и другие помечены deprecated [13]. Headless: `useCourier().inbox` с полями `feeds`, `totalUnreadCount`, `registerFeeds`, `load()`, `listenForUpdates()` (открывает WebSocket). Точные имена методов read/unread на этой странице не перечислены [14].
- **Авторизация.** Бэкенд вызывает `POST /auth/issue-token` с API-ключом и scope `user_id:<id> inbox:read:messages inbox:write:events …`. Без `expires_in` токен никогда не истекает, а SDK его не обновляет, так что `signIn` нужно вызывать заново. Для этого нужен сетевой запрос к Courier на каждую выдачу токена. Пользователь регистрируется при входе. API-ключи свои у каждого окружения [20].

### SuprSend

- **Цены.** Free: 10 000 нотификаций в месяц, 30 дней хранения логов, брендинг SuprSend на ленте. Essentials: $100/мес, 50 000, дальше $2 за 1 000. Business: $250/мес. Self-host только в Enterprise [5].
- **UI.** `@suprsend/react` (drop-in) и `@suprsend/react-core` (headless: `SuprSendFeedProvider`, `useFeedData`, `useFeedClient`, `markAsRead`, `markAsSeen`). Старый `@suprsend/react-inbox` больше не поддерживается [15][16].
- **Авторизация.** JWT ES256 с ключом из дашборда. Claims: `entity_type: "subscriber"`, `entity_id`, `exp`, `iat`, необязательный `scope.tenant_id`. Есть колбэк `refreshUserToken` [21].
- **Окружения.** Workspaces Sandbox, Staging и Production, ключи у каждого свои, есть CLI `suprsend sync` для переноса между ними [22].

## Что проверить на этапе прототипа

1. Novu: сколько хранятся сообщения Inbox на Free.
2. Knock: подтвердить, что брендинг на Free — это только плашка в компоненте и в headless-UI её нет.
3. Оба финалиста: как они считают одно in-app-сообщение одному пользователю. Для Knock это 1 message, для Novu — 1 workflow run на получателя.

## Источники

1. Knock pricing — https://knock.app/pricing
2. Novu pricing — https://novu.co/pricing
3. MagicBell pricing — https://www.magicbell.com/pricing
4. Courier pricing — https://www.courier.com/pricing
5. SuprSend pricing — https://www.suprsend.com/pricing
6. Knock React overview — https://docs.knock.app/in-app-ui/react/overview
   6b. Knock security & authentication — https://docs.knock.app/in-app-ui/security-and-authentication
7. Knock headless feed — https://docs.knock.app/in-app-ui/react/headless/feed
8. Novu Inbox overview — https://docs.novu.co/platform/inbox/overview
9. Novu headless mode и хуки — https://docs.novu.co/platform/inbox/headless-mode, https://docs.novu.co/platform/sdks/react/hooks/use-counts, https://docs.novu.co/platform/sdks/react/hooks/use-notifications
10. Novu prepare for production (HMAC) — https://docs.novu.co/platform/inbox/prepare-for-production
11. MagicBell React — https://www.magicbell.com/docs/libraries/magicbell-react
12. MagicBell React headless — https://magicbell.com/docs/libraries/react-headless (плюс README `@magicbell/react-headless` на unpkg)
13. Courier Inbox overview — https://www.courier.com/docs/platform/inbox/inbox-overview
14. Courier React hooks — https://www.courier.com/docs/sdk-libraries/courier-react-web-hooks
15. SuprSend Inbox React — https://docs.suprsend.com/docs/inbox-react
16. SuprSend React headless — https://docs.suprsend.com/docs/react-headless
17. Knock feeds / socket behavior — https://docs.knock.app/in-app-ui/feeds/overview, https://docs.knock.app/in-app-ui/feeds/socket-behavior
18. Knock users — https://docs.knock.app/concepts/users
19. MagicBell User JWT — https://www.magicbell.com/docs/api/authentication/user
20. Courier Inbox authentication — https://www.courier.com/docs/platform/inbox/authentication
21. SuprSend client authentication — https://docs.suprsend.com/docs/client-authentication
22. SuprSend workspaces — https://docs.suprsend.com/docs/workspaces
23. Novu self-hosting — https://docs.novu.co/community/self-hosting-novu/overview
24. Novu repo (лицензии) — https://github.com/novuhq/novu
