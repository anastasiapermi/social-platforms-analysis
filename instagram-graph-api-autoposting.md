# Instagram Graph API — автопостинг

Официальная документация: https://developers.facebook.com/docs/instagram-platform/content-publishing/
Основные методы: `POST /{ig-id}/media` (создать контейнер), `POST /{ig-id}/media_publish` (опубликовать), `GET /{ig-container-id}?fields=status_code` (статус), `GET /{ig-id}/content_publishing_limit` (лимиты).

## 1. Summary

Instagram Content Publishing API — двушаговый flow через контейнеры: сначала создаётся media container с медиа, затем `media_publish` его публикует. Поддерживает single image/video, REELS, STORIES, карусели до 10 элементов. **Главные блокеры:**
- Требуется **Instagram Professional Account** (Business или Creator), подключённый к Facebook Page.
- Permission `instagram_content_publish` требует **App Review** для Advanced Access.
- **Изображения принимаются только по публичному URL** — для фото у нас обязан быть CDN/storage с публичными ссылками. Для видео есть альтернатива (resumable upload).

**Сложность интеграции: high.**

## 2. Что нужно для запуска

1. Meta-приложение в https://developers.facebook.com, прошедшее App Review для нужных permissions.
2. **Instagram Professional account** (Business или Creator), подключённый к Facebook Page.
3. OAuth flow для пользователя — получение access token с нужными scope.
4. Для фото — хостинг медиа на публичном URL (S3, CDN или собственный сервер). Для видео — альтернативно resumable upload, тогда публичный URL не нужен.

## 3. Авторизация

Документация: https://developers.facebook.com/docs/instagram-platform/

Два варианта API/Login:

| Аспект            | Instagram API with Instagram Login | Instagram API with Facebook Login |
|-------------------|-----------------------------------|-----------------------------------|
| Host              | `graph.instagram.com`             | `graph.facebook.com` + `rupload.facebook.com` |
| Login flow        | Business Login for Instagram      | Facebook Login for Business       |
| Тип access token  | Instagram User access token       | Facebook Page access token        |
| Authorization code TTL | 1 час (одноразовый, подтверждено) | 1 час (одноразовый, подтверждено) |
| Long-lived token TTL | **60 дней** (подтверждено) | 60 дней (через page-token endpoint) |
| Refresh long-lived | `GET /refresh_access_token` продлевает ещё на 60 дней | `GET /me/accounts` для page token |

Передача токена: `Authorization: Bearer <ACCESS_TOKEN>` или query-параметр `access_token=...`.

OAuth flow стандартный Meta OAuth 2.0 (Business Login for Instagram):
1. Redirect на `https://www.instagram.com/oauth/authorize` (или Facebook OAuth для Facebook Login варианта).
2. Получение `code` (TTL **1 час**, одноразовый).
3. Обмен code на short-lived token: `POST https://api.instagram.com/oauth/access_token`.
4. Обмен short-lived на long-lived: `GET https://graph.instagram.com/access_token`. Long-lived действует **60 дней**.
5. Рефреш long-lived: `GET https://graph.instagram.com/refresh_access_token` — продлевает ещё на 60 дней.

## 4. Права / scopes

### Instagram API with Instagram Login

- [`instagram_business_basic`](https://developers.facebook.com/docs/permissions#instagram_business_basic) — базовое чтение профиля.
- [`instagram_business_content_publish`](https://developers.facebook.com/docs/permissions#instagram_business_content_publish) — публикация контента.

### Instagram API with Facebook Login

- [`instagram_basic`](https://developers.facebook.com/docs/permissions#instagram_basic) — базовое чтение.
- [`instagram_content_publish`](https://developers.facebook.com/docs/permissions#instagram_content_publish) — публикация контента.
- [`pages_read_engagement`](https://developers.facebook.com/docs/permissions#pages_read_engagement) — для работы с Page, к которой привязан IG.

### App Review

Оба варианта `*_content_publish` требуют **App Review для Advanced Access**. Без него (Standard Access):
- Можно использовать только в режиме разработки.
- Доступ только тестовым пользователям приложения.

App Review — это формальная процедура Meta с предоставлением:
- Скринкаста use-case (как пользователь подключает аккаунт и постит).
- Объяснения, зачем нужен permission.
- Privacy Policy, Terms of Service.
- Соответствия Platform Policy.

Срок ревью — от нескольких дней до недель, могут отказать.

**Единое приложение Meta на Facebook, Instagram и Threads.** App Review в Meta App Dashboard — одна процедура на всё Meta-семейство: в одном приложении мы запрашиваем permissions для Facebook Pages (`pages_manage_posts` и др.), Instagram (`instagram_content_publish`) и Threads (`threads_content_publish`) сразу, заявка подаётся одна. Это снижает суммарную бюрократию относительно подачи трёх отдельных заявок. **X (Twitter) — отдельная компания, не Meta**: свой dev-портал, свой review, к этому единому пайплайну не относится.

### Page Publishing Authorization (PPA)

Документация: [Get authorized to post or interact as your Page](https://www.facebook.com/help/1939753742723975), [New Authorization for Pages](https://www.facebook.com/business/news/new-authorization-for-pages).

PPA — отдельная процедура Meta по верификации владельца Page. Если IG-аккаунт привязан к такой Page, постить через API нельзя до прохождения PPA одним из админов.

**Кому именно нужен PPA.** Meta не публикует точный порог в числах. Требование триггерится для Pages «с большим охватом» (high-reach), исторически — это:
- Pages новостных организаций, политических субъектов, общественно-значимых публичных фигур.
- Pages с крупной аудиторией (обычно от десятков-сотен тысяч подписчиков, конкретный порог Meta держит закрытым).
- Pages из категорий, которые Meta помечает как чувствительные к дезинформации.

Если PPA триггерится — у админа в Settings Facebook появится prompt «Complete Page Publishing Authorization», без выполнения которого публикация (в том числе через API) для этой Page блокируется. Через API заранее проверить, нужен ли PPA конкретной Page, **нельзя** — клиент узнаёт только из своего UI.

**Как проходить (со стороны клиента).**
1. В личном Facebook-аккаунте админа: Settings → Identity Confirmation → нажать Get Started в «ID Hub».
2. Загрузить документ, удостоверяющий личность (формат — см. требования в самом UI).
3. Подтвердить страну происхождения и включить 2FA.
4. Meta проверяет от нескольких дней до нескольких недель.
5. После одобрения админ снова может публиковать как Page (и API-постинг для связанного IG разблокируется).

**Что это значит для нашего сервиса.** Через API ситуацию ни предсказать, ни обойти нельзя. В UI подключения IG-канала имеет смысл превентивно вывести инфоблок: «если у вас крупная или news-/political-Page, до начала постинга админу нужно пройти Page Publishing Authorization в личных настройках Facebook». При попытке `media_publish` от заблокированного PPA Page API вернёт permission-error — обрабатывать как отдельный тип ошибки в нашем UI и показывать клиенту инструкцию.

## 5. Адресация

Постинг идёт **на свой собственный IG-аккаунт** через его `IG_ID` (Instagram User ID):
```
POST /<IG_ID>/media
POST /<IG_ID>/media_publish
```

Чужие аккаунты постить нельзя — только тот, чей access token используется.

## 6. Постинг текста

«Чисто текстовый» пост в Instagram **невозможен** — Instagram требует медиа. Текст всегда живёт в `caption` (подписи), привязанной к медиа.

Лимит `caption`: **2200 символов**, до 30 хэштегов, до 20 упоминаний `@`. Markdown/HTML не поддерживается.

## 7. Постинг медиа

Документация: https://developers.facebook.com/docs/instagram-platform/content-publishing/

### Поддерживаемые форматы

- **IMAGE** — только JPEG (JPS/MPO не поддерживаются).
- **VIDEO** — legacy media type для постов в фид.
- **REELS** — короткие видео.
- **STORIES** — сторис (24 часа).
- **CAROUSEL** — карусель до 10 фото/видео.

### Flow публикации

Двушаговый: сначала создаётся media-контейнер (`POST /{IG_ID}/media`) с метаданными и медиа, затем — публикация (`POST /{IG_ID}/media_publish` с `creation_id`). Контейнер живёт **24 часа**, после чего expire'ит. Для карусели: создать N контейнеров с `is_carousel_item=true`, создать CAROUSEL-контейнер с массивом `children`, опубликовать. Статус контейнера опрашивается через `GET /{container-id}?fields=status_code` — рекомендация Meta: **раз в минуту, не более 5 минут**. Возможные значения `status_code`: `IN_PROGRESS`, `FINISHED`, `PUBLISHED`, `EXPIRED` (контейнер не опубликован за 24 ч), `ERROR`.

### Передача медиа: варианты инфры

- **Через публичный URL** (`image_url` / `video_url`). Передаём Instagram ссылку, Instagram анонимно скачивает по ней файл. Простой код. **Требует**: клиентские медиа должны лежать на публично доступном хосте (S3 с public bucket, Cloudflare R2, своя статика). Стоимость CDN-трафика, политика хранения, безопасность (медиа доступно по URL). Источник: раздел «Media on a public server» в [Content Publishing](https://developers.facebook.com/docs/instagram-platform/content-publishing/) — прямая цитата: *«We cURL media used in publishing attempts, so the media must be hosted on a publicly accessible server at the time of the attempt»*.
- **Через resumable upload** (`upload_type=resumable` + POST байтов на `rupload.facebook.com/ig-api-upload/...`). Шлём байты файла прямо в HTTP-запрос. **Не требует** публичного хоста. Поддерживает докачку при обрывах. **Ограничения**: только для видео (image альтернативы не имеет — публичный URL обязателен) и только для приложений на Facebook Login for Business. Источник: раздел [«Resumable Upload Session»](https://developers.facebook.com/docs/instagram-platform/content-publishing/#resumable-upload-session).

**Выбор остаётся за решением о приоритетах** и от микса контента: если контент в основном фото — публичный URL обязателен в любом случае; если в основном видео — resumable upload снимает требование к публичному хостингу.

### Особенности карусели

- До 10 элементов.
- Все обрезаются под соотношение первого изображения (по умолчанию 1:1).

## 8. Rate limits

### Publishing rate limit

В живой [документации](https://developers.facebook.com/docs/instagram-platform/content-publishing/) есть **расхождение между двумя разделами** одной и той же страницы:

- Раздел «Rate Limit» вверху: *«Instagram accounts are limited to **100 API-published posts within a 24-hour moving period**. Carousels count as a single post. This limit is enforced on the POST /<IG_ID>/media_publish endpoint»*.
- Раздел «Create a carousel container» ниже: *«Accounts are limited to **50 published posts within a 24-hour period**. Publishing a carousel counts as a single post»*.

Безопасно ориентироваться на **50 постов / 24 ч** как на более жёсткую границу.

Общее:
- Карусель = 1 пост в счёте лимита.
- Лимит действует на `POST /{IG_ID}/media_publish`.
- Текущее использование проверяется через `GET /{IG_ID}/content_publishing_limit` ([документация](https://developers.facebook.com/docs/instagram-platform/reference/instagram-user/content_publishing_limit)).

### Graph API rate limit

Общий лимит Meta Graph API: **200 calls per hour per user** (плавающее окно). Превышение → ошибка с заголовком `X-App-Usage`.

## 9. Особенности

- **К нашему сервису можно подключить только Professional-аккаунт Instagram.** API публикации работает исключительно с аккаунтами типа Professional (зонтичный термин Meta для двух подтипов: **Business** и **Creator**). Личный (Personal) IG — закрыт для API полностью. Если у клиента сейчас личный аккаунт, он не пройдёт наш OAuth, пока не переключится. Переключение бесплатное, делается в самом IG-приложении (Settings → Account type and tools → Switch to Professional Account), обратимо. Прямая цитата из [Instagram Platform Overview](https://developers.facebook.com/docs/instagram-platform/overview): *«To use the APIs, your app users must have an Instagram professional account. An Instagram professional account can be for a business or creator»*. В UI подключения IG-канала нужно явно вывести предупреждение и короткую инструкцию для клиентов с личными аккаунтами.
- **App Review обязателен для prod** — пока не пройдём, постить смогут только люди с ролью на нашем Meta-приложении (Roles → Testers). Лимит **общий для всего приложения**, а не для каждой платформы: одно Meta-приложение покрывает Facebook + Instagram + Threads, и лимит testers считается на приложение целиком. Значит, **до 50 testers суммарно по всем трём платформам** (или **до 500** для приложения, привязанного к Business Manager с Business Verification). Один tester, добавленный в наше приложение, автоматически может тестировать и FB, и IG, и Threads — он занимает один слот, не три. Источник: [Meta App Roles](https://developers.facebook.com/docs/development/build-and-test/app-roles/) — *«Most apps, not linked, can have up to 50 testers. An app that is linked to a Business Manager with Business Verification can have up to a combined total of 500 analytics users and testers»*.
- **Page Publishing Authorization** — может потребовать пользователь, не определяется через API.
- **Текстовых постов нет** — текст только в caption под медиа.
- **Изображения принимаются только по публичному URL** — для фото у нас обязан быть CDN/storage с публичными ссылками. Для видео есть альтернатива — resumable upload (отправка байтов на `rupload.facebook.com`), но она доступна только в варианте с Facebook Login for Business.
- **JPEG only** для image — PNG/HEIC/WEBP не примет.
- **Карусель кадрирует под первый элемент.** Instagram при публикации обрезает все элементы карусели под соотношение сторон **первой** картинки/видео (по умолчанию 1:1). Если в карусели смешаны квадратное + вертикальное + горизонтальное — вертикальному и горизонтальному отрежет края, чтобы стали как первое. **Для нашего UX «черновик → ревью»**: в превью у нас нельзя показывать карусель «как есть», иначе клиент апрувит красивые полные фото, а в Instagram увидит обрезанные и обвинит нас. Превью должно применять то же кадрирование, что Instagram применит после публикации.
- **Нет нативного scheduled publishing через API** — Content Publishing API не поддерживает отложенный постинг. Отложенный постинг строится у нас (хранить в очереди).
- **Stories — 24 часа** — после публикации API не позволяет менять/удалять.

## 10. Что учесть при запуске интеграции

- **App Review** — заложить **минимум 2-4 недели** до prod. Параллельно с разработкой готовить скринкаст, описание use-case, Privacy Policy / ToS, Business Verification.
- **Клиент должен иметь Instagram Professional account** (Business или Creator), подключённый к Facebook Page. Если у клиента личный аккаунт — нужна инструкция в нашем UI: «переключите профиль на Professional, привяжите его к Page, потом подключайте».
- **UX подключения канала**: клиент проходит Meta OAuth → выдаёт нам permissions на свой Instagram. Long-lived токен 60 дней — нужно реализовать автоматический рефреш до истечения.

## Карточка для сводки

| Параметр                                 | Значение                                              |
|------------------------------------------|-------------------------------------------------------|
| Сложность интеграции                     | high                                                  |
| Тип авторизации                          | OAuth user (long-lived 60 дней, требует рефреша)      |
| Требуется app review                     | **да, для Advanced Access** на `instagram_content_publish` |
| Поддержка отложенного постинга в API     | нет (расписание у нас)                                |
| Поддержка карусели/альбома               | да, до 10 элементов                                   |
| Лимит длины текста                       | caption 2200 символов, до 30 хэштегов                 |
| Макс размер медиа                        | image JPEG only; video до 1 ГБ (REELS до 1 ГБ); конкретные лимиты на API reference |
| Rate limit                               | 100 (или 50) постов / 24ч на IG-аккаунт; 200 Graph calls/час на пользователя |
| Платный тариф для повышения лимитов      | нет (но через ads-API есть отдельные продукты)        |
| Блокер для нашего use-case               | App Review (2-4+ недели), требование professional account, для фото — публичный URL обязателен |
| Подходит для «draft → review → publish»  | да — approval у нас; для фото нужен CDN, для видео можно resumable upload |
