# YouTube Data API — автопостинг (Видео и Shorts)

Официальная документация: https://developers.google.com/youtube/v3
Основные методы: `videos.insert` (загрузка и публикация), `thumbnails.set` (превью), `playlistItems.insert` (добавление в плейлист).

## 1. Summary

YouTube Data API v3 — единственный официальный путь для программной загрузки видео. **Текстовых постов нет** (есть Community Posts API, но он только для каналов с ≥500 подписчиками и через Channel API, отдельная история). Shorts — это **обычные видео**, формат определяется YouTube автоматически: квадратное или вертикальное aspect ratio + длительность до 3 минут (для видео, загруженных после 15 октября 2024). **Квота 10 000 units/день по умолчанию**, `videos.insert` стоит **100 units** = **до 100 загрузок/день**.

**Сложность интеграции: medium-high** из-за quota constraints, Google OAuth и resumable upload.

## 2. Что нужно для запуска

1. Google Cloud project (https://console.cloud.google.com) с включённым **YouTube Data API v3**.
2. OAuth 2.0 client credentials (client_id + client_secret).
3. **Verification** Google-приложения: для production access к scope `youtube.upload` нужна **YouTube API Services Compliance Audit**.
4. YouTube-канал у пользователя (создаётся бесплатно).

## 3. Авторизация

Документация: https://developers.google.com/youtube/v3/guides/authentication

- База API: `https://www.googleapis.com/youtube/v3/`.
- Upload endpoint: `https://www.googleapis.com/upload/youtube/v3/videos`.
- Передача токена: `Authorization: Bearer <ACCESS_TOKEN>`.
- OAuth 2.0 Authorization Code Flow (стандартный Google):
  1. Redirect на `https://accounts.google.com/o/oauth2/v2/auth?...`.
  2. Обмен `code` на `access_token` + `refresh_token`.
  3. `access_token` TTL: **3600 сек (1 час)**.
  4. `refresh_token` бессрочный (пока пользователь не отозвал).
- Refresh: стандартный `POST /token` с `grant_type=refresh_token`.

## 4. Права / scopes

| Scope                                                   | Назначение                                |
|---------------------------------------------------------|-------------------------------------------|
| `https://www.googleapis.com/auth/youtube.upload`        | загрузка видео (минимально необходимое)   |
| `https://www.googleapis.com/auth/youtube`               | управление аккаунтом YouTube (шире)       |
| `https://www.googleapis.com/auth/youtube.readonly`      | только чтение                             |
| `https://www.googleapis.com/auth/youtubepartner`        | для YouTube Partner Program (advanced)    |

### YouTube API Services Compliance Audit

`youtube.upload` относится к **restricted scopes** Google. Для production-приложения с реальными пользователями нужно пройти **YouTube API Services Compliance Audit**:
- Описание use-case, scopes.
- Privacy Policy, Terms of Service.
- Demonstration / screencast.
- Application security review.

Срок аудита: **от нескольких недель до нескольких месяцев**.

## 5. Адресация

Видео загружаются в **канал того пользователя, чей access token используется**. Если у пользователя несколько YouTube-каналов (бренд-аккаунты), он выбирает канал на этапе OAuth консента.

## 6. Постинг текста

Чисто текстовых постов в Data API **нет**. Текст всегда — это метаданные видео:
- `snippet.title` — до **100 символов**.
- `snippet.description` — до **5000 символов**, поддерживает URL и таймстэмпы (автораспознаются).
- `snippet.tags` — массив строк, общая длина до 500 символов.

YouTube Community Posts (текст/опрос/изображение прямо в Community tab) **через Data API недоступны** — только через UI YouTube Studio. Существуют упоминания Community Posts API в Channel API, но в публичной документации Data API v3 они не описаны.

## 7. Постинг медиа

Документация: [`videos.insert`](https://developers.google.com/youtube/v3/docs/videos/insert), [`thumbnails.set`](https://developers.google.com/youtube/v3/docs/thumbnails/set), [resumable upload protocol](https://developers.google.com/youtube/v3/guides/using_resumable_upload_protocol).

### Flow загрузки видео

`videos.insert` использует **resumable upload** (обязательно для prod):

1. POST на `/upload/youtube/v3/videos?uploadType=resumable` с метаданными (`snippet`, `status`) — получаем `Location: <UPLOAD_URL>`.
2. PUT байтов на `<UPLOAD_URL>` — можно чанками с `Content-Range`.
3. При обрыве — GET на `<UPLOAD_URL>` с `Content-Range: bytes */<file_size>` возвращает `308 Resume Incomplete` с указанием, сколько уже принято; продолжаем PUT с нужного offset.
4. При успехе — 200 OK с полным `video` resource (включая `id` = YouTube video ID).

### Отложенная публикация

YouTube умеет публиковать видео по расписанию **на своей стороне**, в нашей очереди держать ничего не надо. Как это работает: при загрузке видео ставим его приватным и в параметре `status.publishAt` передаём дату и время будущей публикации (в формате ISO 8601, например `2026-09-01T10:15:30+01:00`). В указанный момент YouTube сам переключает видео из приватного в публичное. Обязательное условие — на момент загрузки видео должно быть именно приватным; если поставить публичным, отложенная публикация работать не будет.

Документация на параметр: [`videos.insert` → `status.publishAt`](https://developers.google.com/youtube/v3/docs/videos/insert).

### Shorts

Отдельного API для Shorts нет — загружаются как обычные видео через `videos.insert`. YouTube автоматически классифицирует видео как Shorts по двум критериям (источник: [Understand three-minute YouTube Shorts](https://support.google.com/youtube/answer/15424877)):

- **Aspect ratio**: квадратное или вертикальное (для обычных вертикальных Shorts — 9:16 рекомендуется, но также подходят и другие вертикальные форматы).
- **Длительность**: до **3 минут** (для видео, загруженных после 15 октября 2024 для standard channels; для Official Artist Channels — после 8 декабря 2025). Раньше лимит был 60 секунд — это устаревшая информация.

Хэштег `#Shorts` в title/description не обязателен, формат определяется по самому файлу.

### Thumbnail

Custom превью устанавливается отдельным вызовом `thumbnails.set?videoId=<VIDEO_ID>` после загрузки видео. Лимит размера thumbnail — 2 MB.

### Спецификации видео

Полные требования: раздел «Constraints» в [`videos.insert`](https://developers.google.com/youtube/v3/docs/videos/insert).

- Контейнер: MP4 (рекомендуется), MOV, AVI, WMV, FLV, 3GPP, WebM, MPEGPS.
- Видео-кодек: H.264 / VP9 / AV1.
- Audio: AAC / Opus.
- Максимальный размер: **256 GB** или **12 часов** (что меньше).
- Резолюция: до 8K, frame rate 60 fps.

## 8. Rate limits (квота)

Документация: https://developers.google.com/youtube/v3/getting-started#quota

### Quota system

YouTube Data API использует «единицы» (units) вместо RPS:
- **Default: 10 000 units / день** на project.
- Reset: midnight **Pacific Time** (PT).
- Превышение → HTTP 403 с `quotaExceeded`.

### Стоимость в units

| Метод               | Cost  |
|---------------------|-------|
| `videos.insert`     | **100** |
| `videos.update`     | 50    |
| `videos.list`       | 1     |
| `videos.rate`       | 50    |
| `thumbnails.set`    | 50    |
| `playlistItems.insert` | 50 |
| `search.list`       | **100** |
| любой запрос        | мин. 1 |

При default-квоте: **до 100 загрузок видео / день** на проект.

### Расширение квоты

Через форму **YouTube API Services Quota Extension Request Form** — нужно обоснование use-case. Срок рассмотрения — недели.

## 9. Особенности

- **`containsSyntheticMedia`** — флаг, которым видео помечается как «содержит сгенерированный или существенно изменённый ИИ-контент» (AI-генерация, voice cloning, deepfake, переозвучка реального человека). Введён YouTube 30 октября 2024. Документация: [Revision History](https://developers.google.com/youtube/v3/revision_history) (запись от 30.10.2024) и [`videos.insert` → `status.containsSyntheticMedia`](https://developers.google.com/youtube/v3/docs/videos/insert). YouTube требует disclosure: если креатор не пометил AI-контент явно, видео могут удалить и наложить санкции на канал. В нашем UI публикации в YouTube надо дать клиенту чекбокс «сгенерировано/отредактировано через ИИ» и при `true` передавать `containsSyntheticMedia=true`.
- **Shorts автоопределяются** — нет специального флага, только формат файла (квадрат/вертикальное + до 3 минут с 15 октября 2024; раньше было 60 сек).
- **Community Posts недоступны** через Data API v3 — только видео. Текстовых обновлений не сделать. Проверено по [API Reference](https://developers.google.com/youtube/v3/docs) — среди 20 типов ресурсов (Activities, Channels, Videos, Playlists, Comments, Subscriptions и др.) нет Community / CommunityPosts эндпоинта.
- **Refresh tokens бессрочные**, но могут быть отозваны Google при неактивности (>6 месяцев) или при изменении пароля пользователя.

## 10. Что учесть при запуске интеграции

- **Quota extension через Compliance Audit** — если планируется >100 загрузок видео в день суммарно по всем клиентам. Заложить **от нескольких недель до нескольких месяцев**. Без аудита квоты по умолчанию хватает на 100 загрузок/день/проект.
- **UX подключения канала**: клиент логинится через Google → если несколько YouTube-каналов, выбирает один → мы сохраняем refresh_token. Access token 1 ч — рефрешим по необходимости, refresh_token бессрочный.

## Карточка для сводки

| Параметр                                 | Значение                                              |
|------------------------------------------|-------------------------------------------------------|
| Сложность интеграции                     | medium-high                                           |
| Тип авторизации                          | Google OAuth 2.0; access 1h, refresh бессрочный       |
| Требуется app review                     | **да, YouTube API Services Compliance Audit** для production |
| Поддержка отложенного постинга в API     | **да, нативно** (`status.publishAt`, требует `private`) |
| Поддержка карусели/альбома               | нет (видео — отдельная единица; Shorts — это просто видео) |
| Лимит длины текста                       | title 100, description 5000, tags 500                 |
| Макс размер медиа                        | до 256 GB или 12 часов                                |
| Rate limit                               | квота **10 000 units/день** (default); `videos.insert` = 100 units → **100 загрузок/день** |
| Платный тариф для повышения лимитов      | нет; quota extension через форму запроса              |
| Блокер для нашего use-case               | Compliance Audit + квота 100 видео/день на проект     |
| Подходит для «draft → review → publish»  | да — approval у нас; есть native scheduled через `publishAt` |
