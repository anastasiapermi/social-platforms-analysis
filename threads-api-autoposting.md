# Threads API — автопостинг

Официальная документация: https://developers.facebook.com/docs/threads/
Основные методы: `POST /{threads-user-id}/threads` (контейнер), `POST /{threads-user-id}/threads_publish` (публикация), `GET /{threads-user-id}/threads_publishing_limit` (лимиты).

## 1. Summary

Threads API — двушаговый flow контейнер→публикация (как Instagram). Поддерживает текст до 500 символов, изображение, видео, карусель **до 20 элементов** (в 2 раза больше Instagram). Уникальные фичи: один topic-tag на пост, link preview через `link_attachment` (только для текстовых постов, до 5 ссылок), GIF через GIPHY. Native scheduled publishing нет.

**Сложность интеграции: medium-high** — то же ревью Meta, что для Instagram/Facebook.

## 2. Что нужно для запуска

1. Meta-приложение в https://developers.facebook.com.
2. **Threads-аккаунт** пользователя.
3. App Review для `threads_content_publish` (Advanced Access).
4. OAuth flow Facebook Login → access token с нужными scope.
5. Хостинг медиа на публичном URL (Meta скачивает по cURL).

## 3. Авторизация

Документация: https://developers.facebook.com/docs/threads/overview/

- База API: `https://graph.threads.net/v1.0` или `https://graph.threads.com/v1.0`.
- Передача токена: `access_token=<TOKEN>` в query или body (стандартный Meta OAuth).
- OAuth flow стандартный Meta:
  1. Facebook Login → User access token.
  2. Long-lived token (60 дней).
  3. Refresh до истечения.

## 4. Права / scopes

| Scope                       | Назначение                                |
|-----------------------------|-------------------------------------------|
| `threads_basic`             | базовое чтение профиля                    |
| `threads_content_publish`   | **публикация постов** (key)               |
| `threads_manage_replies`    | публикация и управление ответами          |
| `threads_read_replies`      | чтение ответов                            |

### App Review

`threads_content_publish` требует **App Review для Advanced Access**. Без него (Standard Access):
- Постить можно только в собственный/тестовый аккаунт.
- Live use невозможен.

**Лимит testers общий для всего Meta-приложения**, а не для платформы. Одно Meta-приложение покрывает Facebook + Instagram + Threads, и testers считаются на приложение целиком: **до 50 testers суммарно по всем трём платформам** (или **до 500** для приложения, привязанного к Business Manager с Business Verification). Один tester в нашем приложении автоматически может тестировать FB, IG и Threads — это один слот, а не три. Источник: [Meta App Roles](https://developers.facebook.com/docs/development/build-and-test/app-roles/) — *«Most apps, not linked, can have up to 50 testers. An app that is linked to a Business Manager with Business Verification can have up to a combined total of 500 analytics users and testers»*.

Процедура и требования те же, что для Instagram/Facebook.

## 5. Адресация

Постинг идёт в **собственный** Threads-профиль через `THREADS_USER_ID`:
```
POST /<THREADS_USER_ID>/threads
POST /<THREADS_USER_ID>/threads_publish
```

Чужие профили постить нельзя.

## 6. Постинг текста

Документация: https://developers.facebook.com/docs/threads/posts/

Текстовый пост создаётся через `POST /{THREADS_USER_ID}/threads` с `media_type=TEXT` и параметром `text`. Параметры:
- `media_type` — `TEXT`, `IMAGE`, `VIDEO` (для single) или `CAROUSEL` (для карусели).
- `text` — текст поста.
- `link_attachment` — URL для link preview (только для `TEXT`).
- `gif_attachment` — `{"gif_id":"...","provider":"GIPHY"}` (только для `TEXT`).
- `topic_tag` — тематический тег.

### Лимиты

Документация: https://developers.facebook.com/docs/threads/posts/

- **Текст: 500 символов** (эмодзи считаются как байты UTF-8).
- Topic tag: 1-50 символов, без пробелов/`.`/`&`/`@`/`!`/`?`/`,`/`;`/`:`. Только один tag на пост.
- Ссылки: **до 5 уникальных URL** в посте (с 22 декабря 2025). Считаются URL в `text` + `link_attachment`, дубли = 1.

## 7. Постинг медиа

Документация: https://developers.facebook.com/docs/threads/posts/

### Flow публикации

**Single image / video** — двушаговый: `POST /{TUID}/threads` создаёт контейнер с `media_type=IMAGE`/`VIDEO` и `image_url` / `video_url`, затем `POST /{TUID}/threads_publish` с `creation_id` — публикация. Между шагами рекомендуется подождать **~30 секунд**, чтобы контейнер успел обработаться.

**Карусель** — трёхшаговый: создать по контейнеру на каждый элемент с `is_carousel_item=true`, затем создать CAROUSEL-контейнер с массивом `children`, потом publish. Минимум 2 элемента, максимум 20. Карусель = 1 пост в счёте лимита.

### Спецификации медиа

Полные требования: [Media Specifications](https://developers.facebook.com/docs/threads/posts/) в документации Threads.

**Image:** JPEG / PNG, до **8 MB**, aspect ratio до 10:1, ширина 320-1440 px, sRGB.

**Video:** MOV / MP4 (H264 или HEVC, AAC audio, progressive scan), 23-60 FPS, до 1920 px по горизонтали, aspect ratio 0.01:1 — 10:1 (рекомендуется 9:16), bitrate до 100 Mbps, длительность до **5 минут**, файл до **1 GB**.

## 8. Rate limits

### Posting

- **250 опубликованных постов / 24h** (rolling window) на профиль.
- **1000 ответов / 24h**.
- Карусель = 1 пост.

Текущее использование можно проверить через `GET /{THREADS_USER_ID}/threads_publishing_limit?fields=quota_usage,config` — [документация](https://developers.facebook.com/docs/threads/troubleshooting/).

### API call rate limit

Threads-специфическая формула: `Calls per 24h = 4800 * Number of Impressions`, минимум impressions = 10 → минимум **48 000 API-calls / 24h**.

Дополнительно: `total_cputime = 720 000 * impressions`, `total_time = 2 880 000 * impressions`.

## 9. Особенности

- **Текст до 500 символов** — самый жёсткий лимит среди исследуемых платформ (для сравнения: Telegram 4096, X 280, MAX 4000).
- **30-секундная пауза** перед `threads_publish` — иначе риск ошибки/некорректной обработки.
- **Link preview только для TEXT** — для image/video/carousel автоопределение ссылок не работает.
- **Максимум 5 ссылок** в посте (с 22.12.2025) — у нас validation на этапе approval.
- **GIF только через GIPHY** — нужно подключение к их API для выбора, ID передаётся в `gif_attachment`.
- **Один topic-tag** — только первый валидный `#tag` в тексте берётся как topic. Остальные `#` — просто текст.
- **Карусель до 20 элементов** — в 2 раза больше Instagram, выгодно для длинных подборок.
- **Native scheduled publishing нет** — отложенное у нас.
- **App Review необходим** — те же требования, что для Instagram/Facebook.
- **Только собственный профиль** — постить от имени другого пользователя нельзя.
- **Два host'а** (`graph.threads.net` или `graph.threads.com`) — функционально эквивалентны.

## 10. Что учесть при запуске интеграции

- **App Review** для `threads_content_publish` — заложить **2-4+ недели**. Заявка идёт параллельно с Instagram/Facebook в рамках одного Meta-приложения.
- **UX подключения канала**: клиент проходит Facebook Login → даёт согласие → мы получаем `threads_user_id` и access token. Long-lived токен 60 дней — нужно реализовать рефреш до истечения.

## Карточка для сводки

| Параметр                                 | Значение                                              |
|------------------------------------------|-------------------------------------------------------|
| Сложность интеграции                     | medium-high                                           |
| Тип авторизации                          | OAuth user (long-lived 60 дней)                       |
| Требуется app review                     | **да, для Advanced Access** на `threads_content_publish` |
| Поддержка отложенного постинга в API     | нет (расписание у нас)                                |
| Поддержка карусели/альбома               | да, **до 20 элементов** (вдвое больше Instagram)      |
| Лимит длины текста                       | **500 символов** (самый жёсткий среди исследуемых)    |
| Макс размер медиа                        | image 8 MB; video 1 GB / 5 мин                        |
| Rate limit                               | 250 постов / 24h; 1000 ответов / 24h; API calls 4800·impressions |
| Платный тариф для повышения лимитов      | нет                                                   |
| Блокер для нашего use-case               | App Review + 500-символьный лимит может убить часть контента |
| Подходит для «draft → review → publish»  | да — approval у нас; нужна CDN для медиа              |
