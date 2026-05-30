# LinkedIn Posts API — автопостинг

Официальная документация: https://learn.microsoft.com/en-us/linkedin/marketing/community-management/shares/posts-api
Основные методы: `POST /rest/posts` (универсальный), Images API, Videos API, Documents API, MultiImage API (для нескольких изображений в organic).

## 1. Summary

LinkedIn Posts API — единый эндпоинт `POST /rest/posts` принимает любой тип поста (текст, изображение, видео, документ, статья, опрос, MultiImage, празднование). Автор задаётся через URN: `urn:li:person:{id}` для личных постов или `urn:li:organization:{id}` для постов от страницы компании. Карусели — только sponsored, MultiImage и Poll — только organic.

**Сложность интеграции: medium-high** — версионирование API, специфичный Restli-протокол, отдельные API для аплоада медиа.

## 2. Что нужно для запуска

1. Зарегистрированное LinkedIn-приложение (https://www.linkedin.com/developers/apps).
2. Согласование use-case с LinkedIn (часть permissions доступна только Marketing Partners или после явного approval).
3. OAuth 2.0 flow для access token с нужными scopes.
4. Для постинга от компании — у пользователя роль на Page: **ADMINISTRATOR**, **DIRECT_SPONSORED_CONTENT_POSTER** или **CONTENT_ADMIN**.
5. Версия API: указывается в заголовке `Linkedin-Version: YYYYMM` (например `202605`). Версии депрекейтят раз в год.

## 3. Авторизация

Документация: https://learn.microsoft.com/en-us/linkedin/shared/authentication/authentication

- База API: `https://api.linkedin.com/rest/`.
- Передача токена: `Authorization: Bearer <ACCESS_TOKEN>`.
- Обязательные заголовки на каждом запросе:
  - `Linkedin-Version: 202605` (текущая на момент написания)
  - `X-Restli-Protocol-Version: 2.0.0`
  - `Content-Type: application/json`
- OAuth 2.0 (Authorization Code Flow):
  1. Redirect на `https://www.linkedin.com/oauth/v2/authorization?response_type=code&client_id=...&redirect_uri=...&scope=...&state=...`.
  2. Обмен `code` на access token через `POST https://www.linkedin.com/oauth/v2/accessToken`.
  3. **Access token TTL — 60 дней** (`expires_in=5184000`). Источник: [Authorization Code Flow](https://learn.microsoft.com/en-us/linkedin/shared/authentication/authorization-code-flow) — *«Currently, all access tokens are issued with a 60-day lifespan»*.
  4. **Authorization code TTL — 30 минут**, должен быть использован сразу после получения, иначе истекает. Источник: там же — *«For security reasons, the authorization code has a 30-minute lifespan and must be used immediately»*.
  5. **Refresh token TTL — 365 дней**, доступен **только для approved Marketing Developer Platform (MDP) partners**. Источник: [Refresh Tokens with OAuth 2.0](https://learn.microsoft.com/en-us/linkedin/shared/authentication/programmatic-refresh-tokens) — *«By default, access tokens are valid for 60 days and programmatic refresh tokens are valid for a year»*. TTL refresh-токена **не сбрасывается** при использовании (отсчитывается от первой выдачи), на 365-й день клиент должен заново пройти OAuth.

## 4. Права / scopes

| Scope                    | Назначение                                                |
|--------------------------|-----------------------------------------------------------|
| `w_member_social`        | публикация от имени пользователя                          |
| `r_member_social`        | чтение постов пользователя — restricted, по approval      |
| `w_organization_social`  | публикация от имени организации/Page (требует роли на Page) |
| `r_organization_social`  | чтение постов организации                                 |

Дополнительно (для медиа):
- `r_basicprofile` — базовые данные.
- `r_liteprofile` / `r_emailaddress` — для базового OAuth.

### App Review / approval

- Базовые scopes доступны при стандартной регистрации приложения.
- `w_organization_social`, `r_organization_social` требуют принадлежности к программе **LinkedIn Marketing Developer Platform (LMDP)**. Подача заявки и условия доступа: https://learn.microsoft.com/en-us/linkedin/marketing/increasing-access.
- Некоторые продвинутые API (Lead Sync, Matched Audiences) — только для Marketing Partners.

## 5. Адресация

Поле `author` в теле POST:
- `urn:li:person:{member_id}` — пост от пользователя (нужен `w_member_social`).
- `urn:li:organization:{org_id}` — пост от страницы компании (нужен `w_organization_social` + роль на Page).

`urn:li:share:...` / `urn:li:ugcPost:...` возвращаются как ID созданного поста (в заголовке `x-restli-id`).

> **Note:** `ugcPosts` API устаревший, заменён Posts API. Старые URN формата `ugcPost` ещё валидны.

## 6. Постинг текста

Документация: [Posts API](https://learn.microsoft.com/en-us/linkedin/marketing/community-management/shares/posts-api).

Публикация — `POST /rest/posts` с обязательными заголовками `Linkedin-Version` (YYYYMM) и `X-Restli-Protocol-Version: 2.0.0`. Тело — JSON с `author` (URN), `commentary` (текст), `visibility`, `distribution`, `lifecycleState`. Ответ: HTTP 201, URN созданного поста возвращается в заголовке `x-restli-id`.

### Форматирование (Little Text Format)

LinkedIn использует свой формат разметки для упоминаний и форматирования:
- Упоминание: `{member URN}` в `commentary` через специальные плейсхолдеры.
- Hashtags и ссылки распознаются автоматически.
- Markdown/HTML не поддерживается.

Документация формата: https://learn.microsoft.com/en-us/linkedin/marketing/community-management/shares/little-text-format

## 7. Постинг медиа

Документация: [Images API](https://learn.microsoft.com/en-us/linkedin/marketing/community-management/shares/images-api), [Videos API](https://learn.microsoft.com/en-us/linkedin/marketing/community-management/shares/videos-api), [Documents API](https://learn.microsoft.com/en-us/linkedin/marketing/community-management/shares/documents-api), [Posts API](https://learn.microsoft.com/en-us/linkedin/marketing/community-management/shares/posts-api).

### Типы контента, доступные нам

**Наш use-case — только organic-постинг** (обычные посты бесплатно в feed автора, без таргетинга и платного продвижения). Sponsored — это рекламные посты с targeting'ом, требуют Marketing/Ads API + рекламного бюджета; не наша история. Документация по типам: [Posts API](https://learn.microsoft.com/en-us/linkedin/marketing/community-management/shares/posts-api).

Для organic нам доступны:

- **Text-only** — просто текст.
- **Image** (single) — одна картинка с подписью.
- **Video** — видео с подписью.
- **Document** — PDF/документ с подписью.
- **Article** — длинный пост-статья.
- **MultiImage** — несколько картинок в одном посте (LinkedIn-аналог карусели; только фото, без видео).
- **Poll** — пост-опрос.
- **Celebration** — поздравительный пост (день рождения коллеги, продвижение и т.п.).

**Недоступно нам** (только для sponsored / Ads API): **Carousel** (рекламная карусель). Если клиент хочет «карусель» — даём ему MultiImage как замену.

## 8. Rate limits

Документация: [Rate Limiting](https://learn.microsoft.com/en-us/linkedin/shared/api-guide/concepts/rate-limits), [Increasing Access](https://learn.microsoft.com/en-us/linkedin/marketing/increasing-access).

### Two-tier rate limit

| Уровень        | Описание                                                |
|----------------|---------------------------------------------------------|
| **Application**| Общее количество запросов приложения / сутки UTC        |
| **Member**     | Запросов одним member-токеном / сутки                   |

- Reset: **midnight UTC** каждый день.
- 429 при превышении.
- Email-alert при 75% использования (с задержкой 1-2 часа).

### Конкретные лимиты для Community Management API

Задокументированы на странице [Increasing Access](https://learn.microsoft.com/en-us/linkedin/marketing/increasing-access), раздел «How to Upgrade your Access Tier»:

| Tier            | Application (per app / 24h) | Member (per member / 24h) | BATCH_GET-методы | Webhook for Social Actions |
|-----------------|-----------------------------|---------------------------|------------------|----------------------------|
| **Development** | **500 calls**               | **100 calls**             | **запрещены**    | **выключен**               |
| **Standard**    | без ограничений             | без ограничений           | разрешены        | включён                    |

Прямая цитата: *«All APIs: 500 API calls for an app for 24 hrs, 100 API calls per member of an App for 24 hrs»*. То есть в Development tier мы упрёмся в потолок очень быстро при подключении даже нескольких боевых клиентов — Development подходит для разработки/тестирования, но **не для production**. До production надо успеть подать заявку на Standard tier.

## 9. Особенности

- **Роли на Page** — для company-постов пользователь должен быть ADMINISTRATOR / DIRECT_SPONSORED_CONTENT_POSTER / CONTENT_ADMIN на соответствующей странице.
- **Marketing Developer Platform** — `w_organization_social` requires participation in LMDP. Это **дополнительный approval** на стороне LinkedIn.
- **Multi-step media upload** — Images/Videos/Documents API: initializeUpload → PUT → finalizeUpload → URN. Не одношаговый, кешировать URN.
- **Нет нативного scheduled publishing** в Posts API. Для отложенного постинга — наша очередь. (Существует marketing-API для ads scheduling, но это другая история.)
- **Little Text Format** для упоминаний и форматирования — нестандартный, требует подготовки текста на нашей стороне.
- **`commentary` лимит — 3000 символов** (унаследовано от legacy UGC Post API). Прямая цитата из [UGC Post API](https://learn.microsoft.com/en-us/linkedin/marketing/community-management/shares/ugc-post-api): *«NOTE: The maximum length of the text of a UGC Post is 3000 characters»*. На странице нового [Posts API](https://learn.microsoft.com/en-us/linkedin/marketing/community-management/shares/posts-api) для поля `commentary` лимит явно не повторён, но Posts API — просто новый интерфейс к той же сущности поста, поэтому лимит почти наверняка тот же.

## 10. Что учесть при запуске интеграции

- **Заявка в Community Management API (LMDP)** — заложить **1-2 недели** для Development tier, дальше отдельный цикл для Standard. Готовить описание use-case на сайте, Privacy Policy / ToS.
- **12-месячное окно Development tier** — за год после одобрения нужно дотащить интеграцию и подать на Standard, иначе доступ могут отозвать.
- **UX подключения канала**: клиент проходит OAuth 2.0 → выбирает Person (личный профиль) или Organization (Page, если есть админ-права) → сохраняем привязку. Access token 60 дней — нужен рефреш до истечения.

## Карточка для сводки

| Параметр                                 | Значение                                              |
|------------------------------------------|-------------------------------------------------------|
| Сложность интеграции                     | medium-high                                           |
| Тип авторизации                          | OAuth 2.0 (Authorization Code Flow)                   |
| Требуется app review                     | да — LinkedIn Marketing Developer Platform для `w_organization_social` |
| Поддержка отложенного постинга в API     | нет                                                   |
| Поддержка карусели/альбома               | да: MultiImage (organic, только фото) или Carousel (только sponsored) |
| Лимит длины текста                       | **3000 символов** для `commentary` (унаследовано от legacy [UGC Post API](https://learn.microsoft.com/en-us/linkedin/marketing/community-management/shares/ugc-post-api), на странице нового Posts API явно не повторено) |
| Макс размер медиа                        | определяется отдельными Images/Videos/Documents API (видео ресамабельный) |
| Rate limit                               | Community Management API Development: **500 calls/app/24h** + **100 calls/member/24h**; Standard tier — без ограничений |
| Платный тариф для повышения лимитов      | нет (через LinkedIn Marketing partners есть отдельные программы) |
| Блокер для нашего use-case               | LMDP approval для company-постов; sponsored vs organic ограничения |
| Подходит для «draft → review → publish»  | да — approval у нас; native scheduled нет             |
