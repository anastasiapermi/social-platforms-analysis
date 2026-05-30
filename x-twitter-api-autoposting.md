# X (Twitter) API v2 — автопостинг

Официальная документация: https://docs.x.com/x-api
Основные методы: `POST /2/tweets` (создание/редактирование поста), `POST /2/media/upload` (загрузка медиа), `DELETE /2/tweets/{id}` (удаление).

## 1. Summary

X API v2 даёт компактный единый эндпоинт `POST /2/tweets` для создания поста (тред, цитата, опрос, медиа) или редактирования. Аутентификация — OAuth 2.0 user token. **Главный коммерческий блокер: pay-per-usage модель с покупкой кредитов**, не фиксированные tier'ы. Цена per-request зависит от эндпоинта. **Quote tweets доступны только на Enterprise**.

**Сложность интеграции: medium**, но **финансово high** для частого постинга.

## 2. Что нужно для запуска

1. Зарегистрированный X Developer App (https://console.x.com).
2. **Покупка кредитов** для production — биллинг pay-per-usage.
3. OAuth 2.0 настройка нашего app в X Developer Console (`client_id`, `client_secret`, `redirect_uri`, `scopes`) — см. §3.
4. OAuth-подключение каждого клиента (Authorization Code Flow with PKCE) — см. §3.

## 3. Авторизация

Документация: https://docs.x.com/fundamentals/authentication/overview, OAuth 2.0 flow — https://docs.x.com/fundamentals/authentication/oauth-2-0/user-access-token.

### OAuth-настройка нашего app в X Developer Console

Разовая настройка нашего сервиса (один раз на все будущие подключения клиентов). Четыре параметра:

- `client_id` — публичный идентификатор нашего app, выдаёт X. С ним мы редиректим клиента на X-форму логина (он становится частью URL).
- `client_secret` — приватный пароль нашего app, выдаёт X. Используется при обмене `code` на токен.
- `redirect_uri` — URL на нашей стороне, куда X вернёт клиента после того, как он одобрит permissions. Должен быть **заранее прописан в X Developer Console** — X отказывает в OAuth, если callback не совпадает.
- `scopes` — список разрешений, которые мы запрашиваем у клиента. Для постинга: `tweet.write` (создание постов), `media.write` (загрузка медиа), `offline.access` (без этого X не выдаёт refresh-токен).

### OAuth flow подключения клиента

Происходит **на каждого клиента** при первом подключении. Используется **OAuth 2.0 с PKCE** (Authorization Code Flow with Proof Key for Code Exchange):

1. Клиент жмёт «Подключить X» в нашем UI.
2. Мы редиректим его на `https://x.com/i/oauth2/authorize?response_type=code&client_id=...&scope=...&state=...&code_challenge=...&code_challenge_method=S256`.
3. Клиент логинится у X (мы пароль не видим) и подтверждает permissions, которые мы запросили в scopes.
4. X редиректит клиента на наш `redirect_uri` с `code`.
5. Наш backend меняет `code` на пару `access_token` + `refresh_token` через `POST /2/oauth2/token` (`auth_code` живёт **30 секунд** после получения — успеть обменять). `refresh_token` приходит только если в scopes указан `offline.access`.
6. Сохраняем эту пару в нашей БД, привязываем к клиенту.

Дальше все вызовы постинга идут под `access_token` этого клиента — X понимает, что публикуем от его имени, а не от нашего.

**TTL токенов** (по живой документации):
- `access_token` — **2 часа**. Прямая цитата: *«By default, the access token you create through the Authorization Code Flow with PKCE will only stay valid for two hours unless you've used the offline.access scope»*.
- `refresh_token` — выдаётся только при scope `offline.access`, используется для получения нового `access_token` без повторного OAuth у клиента.

## 4. Права / scopes

Документация: [OAuth 2.0 Authorization Code Flow with PKCE — Scopes](https://docs.x.com/fundamentals/authentication/oauth-2-0/authorization-code).

| Scope                  | Назначение                                |
|------------------------|-------------------------------------------|
| `tweet.read`           | чтение постов                             |
| `tweet.write`          | **создание/редактирование постов**        |
| `users.read`           | базовые данные пользователя               |
| `offline.access`       | refresh tokens                            |
| `media.write`          | загрузка медиа                            |

### App approval

- Принудительного App Review как у Meta нет, но платформа может банить за нарушение Terms.
- Доступ к API даёт сразу после создания app и пополнения кредитного баланса в Developer Console (см. §8 «Биллинг»).

## 5. Адресация

Пост идёт в timeline **того пользователя**, чей access token используется. Чужие аккаунты постить нельзя.

Дополнительные опции:
- `community_id` — пост в Community.
- `direct_message_deep_link` — пост-ссылка для перехода в DM.

## 6. Постинг текста

Документация: [`POST /2/tweets`](https://docs.x.com/x-api/posts/create-post).

Текстовый пост создаётся через `POST /2/tweets` с JSON-телом `{"text": "..."}`. Ответ — HTTP 201 с `data.id` (Tweet ID) и `data.text`.

### Лимит длины

- **280 символов** — стандартный лимит из живой документации (https://docs.x.com/fundamentals/counting-characters).
- Нестандартный счёт характеров:
  - Эмодзи = **2 символа** каждое (включая family-секвенции `👨‍👩‍👧‍👦` — тоже 2).
  - CJK (китайский / японский / корейский) = **2 символа** (то есть 140 CJK-символов).
  - **URL = 23 символа** независимо от длины (t.co shortener).
  - Авто-`@mention` в начале reply — **не считается**.
  - Медиа = 0 символов.
- Кодировка UTF-8 NFC (Unicode Normalization Form C).
- Рекомендуется библиотека `twitter-text` для подсчёта.

> 25 000-символьная функция для X Premium-подписчиков **не упомянута** в текущей API-документации (возможно работает только в web-клиенте).

Markdown/HTML не поддерживается. Хэштеги и упоминания автоопределяются.

## 7. Постинг медиа

Документация: [Media](https://docs.x.com/x-api/media/introduction), [`POST /2/media/upload`](https://docs.x.com/x-api/media/quickstart/media-upload-chunked).

### Flow публикации

Двушаговый: сначала загрузить медиа в `POST /2/media/upload` и получить `media_id`, затем передать массив `media_ids` в `POST /2/tweets`.

**Маленькие файлы** — одношаговый upload (multipart `media` + `media_category`).

**Видео и большие файлы** — chunked upload (`command=INIT` → `command=APPEND` для каждого chunk → `command=FINALIZE` → polling `command=STATUS` до `state=succeeded`).

Возможные значения `media_category`: `tweet_image`, `tweet_video`, `tweet_gif`, `amplify_video` (для длинных видео ads).

### Лимиты в одном посте

До **4 медиа** в одном посте. Микс: 1 GIF **или** 1 видео **или** до 4 фото. GIF + видео в одном посте нельзя.

### Спецификации медиа

**Image:** JPG, PNG, GIF, WEBP. Max **5 MB**.

**GIF:** Max **15 MB**.

**Video:** MP4, MOV. Max **512 MB**. Max длительность — **140 секунд (2:20)** по advanced constraint в [Best Practices](https://docs.x.com/x-api/media/quickstart/best-practices). Для аккаунтов с X Premium-подпиской разрешение playback — до 1080p (про длительность в живой документации API ничего не задокументировано).

## 8. Rate limits

Документация: https://docs.x.com/x-api/fundamentals/rate-limits и https://docs.x.com/x-api/fundamentals/post-cap (Usage and Billing).

### Биллинг — pay-per-usage (актуально по живой документации)

X API v2 перешёл на **pay-per-usage** модель. Фиксированных tier'ов Free/Basic/Pro **больше нет** в публичной документации.

- **Credit-based**: покупаешь кредиты заранее, они списываются по мере использования.
- **Per-request pricing**: разные эндпоинты стоят разное; цены смотреть в Developer Console.
- **Real-time tracking**: в Developer Console.
- Для write-операций (создание постов) тарификация per-request.

Дополнительные опции:
- **Set budgets** — лимиты расходов через Developer Console.
- **Monitor alerts** — нотификации перед достижением порога.
- **Enterprise** — кастомные цены, volume discount, dedicated support.

### Per-endpoint rate limits (живая выборка из документации)

| Эндпоинт                                  | Per App         | Per User        |
|-------------------------------------------|-----------------|-----------------|
| `POST /2/tweets`                          | **10 000/24h**  | **100/15min**   |
| `DELETE /2/tweets/:id`                    | —               | 50/15min        |
| `GET /2/tweets`                           | 3 500/15min     | 5 000/15min     |
| `GET /2/tweets/:id`                       | 450/15min       | 900/15min       |
| `POST /2/dm_conversations/...`            | 1 440/24h       | 15/15min, 1 440/24h |
| `POST /2/users/:id/likes`                 | —               | 50/15min, 1 000/24h |
| `POST /2/users/:id/retweets`              | —               | 50/15min        |
| `POST /2/lists`                           | —               | 300/15min       |

Окно — 15 минут или 24 часа. Полная таблица: https://docs.x.com/x-api/fundamentals/rate-limits.

## 9. Особенности

- **Pay-per-usage модель** — нет фиксированных tier'ов. Бюджетирование зависит от объёма постинга и эндпоинтов; следить за tracker'ом в Developer Console.
- **Quote tweets только Enterprise** — `quote_tweet_id` отвергается на pay-per-usage.
- **280-символьный лимит** с нестандартным счётом: эмодзи и CJK = 2 chars, URL фиксированно 23 chars.
- **Native scheduled publishing отсутствует в API v2** — отложенный постинг через нашу очередь. (X UI и партнёрские tools поддерживают, но не v2 API напрямую.)
- **Community posts** через `community_id`, требует членства в community.

## 10. Что учесть при запуске интеграции

- **Покупка кредитов** через Developer Console — обязательна для production (модель pay-per-usage, не подписка). Бюджет планируется заранее по ожидаемому числу постов клиентов в месяц.
- **UX подключения канала**: клиент проходит OAuth 2.0 PKCE → выдаёт нам permissions (`tweet.write` + `media.write` + `offline.access` для рефреша). Access token истекает через **2 часа**, нужен автоматический рефреш через `refresh_token`.

## Карточка для сводки

| Параметр                                 | Значение                                              |
|------------------------------------------|-------------------------------------------------------|
| Сложность интеграции                     | medium (но финансово high — pay-per-usage credits)    |
| Тип авторизации                          | OAuth 2.0 PKCE, access token 2 часа, refresh через `offline.access` scope |
| Требуется app review                     | нет формального; нужна **покупка кредитов**           |
| Поддержка отложенного постинга в API     | нет (только наша очередь)                             |
| Поддержка карусели/альбома               | до 4 медиа в посте (фото / 1 GIF / 1 видео)           |
| Лимит длины текста                       | **280** weighted; эмодзи/CJK=2, URL=23 (t.co)         |
| Макс размер медиа                        | image 5 MB, GIF 15 MB, video 512 MB / до 140 сек        |
| Rate limit                               | per-endpoint: `POST /2/tweets` 10 000/24h app, 100/15min user           |
| Платный тариф для повышения лимитов      | **да, обязательный**: pay-per-usage credits; кастом — Enterprise |
| Блокер для нашего use-case               | **pay-per-usage**; quote tweets только Enterprise; нет native scheduling |
| Подходит для «draft → review → publish»  | да — approval у нас; всё остальное у нас же           |
