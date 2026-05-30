# Telegram Bot API — автопостинг

Официальная документация: https://core.telegram.org/bots/api и FAQ https://core.telegram.org/bots/faq
Основные методы: `sendMessage`, `sendPhoto`, `sendVideo`, `sendAnimation`, `sendAudio`, `sendVoice`, `sendDocument`, `sendVideoNote`, `sendMediaGroup`.

## 1. Summary

Самый простой в интеграции механизм автопостинга среди исследуемых площадок. Авторизация — статический bot-токен, постинг полностью stateless. Поддерживает текст, медиа, альбомы 2-10 элементов (с миксом фото+видео), inline-кнопки, реакции, опросы. **Архитектура: каждый наш клиент создаёт свой бот в @BotFather** — лимиты считаются per-bot, изолированно для каждого клиента.

**Сложность интеграции: low.**

## 2. Что нужно для запуска

1. Бот, созданный через `@BotFather` (команда `/newbot`).
2. Bot-токен в секрет-сторе.
3. Канал (или группа), в который бот добавлен **администратором** с правом `can_post_messages`.
4. `chat_id` целевого канала (`@username` для публичных, числовой `-100xxxxxxxxxx` для приватных).

## 3. Авторизация

Документация: https://core.telegram.org/bots/api#authorizing-your-bot и https://core.telegram.org/bots/api#making-requests

- Токен формата `123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11` получается у `@BotFather`.
- Все запросы — HTTPS на `https://api.telegram.org/bot<TOKEN>/<METHOD>`.
- Форматы тела: query string, `application/x-www-form-urlencoded`, `application/json`, `multipart/form-data`.
- Никакого OAuth, токен — единственный секрет. Отзыв — `/revoke` у BotFather.

## 4. Права / scopes

Документация: https://core.telegram.org/bots/api#chatadministratorrights

Гейт для постинга в канал — одно право администратора:

> `can_post_messages` — True, if the administrator can post messages in the channel, approve suggested posts, or access channel statistics; for channels only

Дополнительные опциональные права:
- `can_edit_messages` — редактировать сообщения и пинить (только каналы).
- `can_delete_messages` — удалять чужие сообщения.
- `can_manage_direct_messages` — управлять direct-messages чатом канала.
- `can_post_stories` / `can_edit_stories` / `can_delete_stories` — для Stories.

Назначение прав — вручную в Telegram UI через настройки канала. Программного API нет (бот добавляется админом владельцем канала).

## 5. Постинг текста

Документация: https://core.telegram.org/bots/api#sendmessage

`sendMessage`: обязательные `chat_id` + `text` (1-4096 символов).

Параметры форматирования:
- `parse_mode`: `"HTML"` или `"MarkdownV2"` (https://core.telegram.org/bots/api#formatting-options).
- `disable_web_page_preview`, `disable_notification`, `protect_content`.

### HTML
```
<b>жирный</b> <i>курсив</i> <u>подчёрк</u> <s>зачёрк</s>
<span class="tg-spoiler">спойлер</span>
<a href="https://example.com">ссылка</a>
<a href="tg://user?id=123">упоминание по id</a>
<code>inline</code>
<pre><code class="language-python">code block</code></pre>
<blockquote>цитата</blockquote>
```

### MarkdownV2
Спецсимволы, требующие экранирования: `_ * [ ] ( ) ~ ` > # + - = | { } . !`

## 6. Постинг медиа

Документация по методам:
- https://core.telegram.org/bots/api#sendphoto
- https://core.telegram.org/bots/api#sendvideo
- https://core.telegram.org/bots/api#sendanimation
- https://core.telegram.org/bots/api#sendaudio
- https://core.telegram.org/bots/api#sendvoice
- https://core.telegram.org/bots/api#senddocument
- https://core.telegram.org/bots/api#sendvideonote
- https://core.telegram.org/bots/api#sendmediagroup
- https://core.telegram.org/bots/api#sending-files
- self-hosted Bot API server: https://core.telegram.org/bots/api#using-a-local-bot-api-server

### Методы и лимиты подписи

| Метод           | Содержимое            | Подпись   |
|-----------------|-----------------------|-----------|
| `sendPhoto`     | jpg / png / webp      | 1024 симв.|
| `sendVideo`     | mp4 (h264+aac)        | 1024      |
| `sendAnimation` | gif / mp4 без звука   | 1024      |
| `sendAudio`     | mp3 / m4a с тегами    | 1024      |
| `sendVoice`     | ogg / opus            | 1024      |
| `sendDocument`  | любой файл            | 1024      |
| `sendVideoNote` | квадратное видео      | -         |

### Альбом `sendMediaGroup`

- 2-10 элементов в одной группе.
- Типы: `InputMediaAudio`, `InputMediaDocument`, `InputMediaLivePhoto`, `InputMediaPhoto`, `InputMediaVideo`.
- Documents и audio группируются **только сами с собой**; photo + video + live photo миксуются.
- Подпись задаётся на первом элементе.

### Загрузка файлов — три способа

| Способ        | Фото      | Остальное |
|---------------|-----------|-----------|
| `file_id`     | без лимита| без лимита|
| URL           | 5 MB      | 20 MB     |
| multipart     | 10 MB     | 50 MB     |

self-hosted Bot API server поднимает лимит до **2000 MB** (прямая цитата из документации Telegram: *«Upload files up to 2000 MB»*).

## 7. Rate limits

Документация: https://core.telegram.org/bots/faq#broadcasting-to-users
Платный тариф: https://core.telegram.org/bots/faq#how-can-i-message-all-of-my-bot-39s-subscribers-at-once

| Контекст                    | Лимит                                |
|-----------------------------|--------------------------------------|
| Один приватный чат          | ~1 сообщение / сек                   |
| Одна группа                 | 20 сообщений / мин                   |
| Глобально (bulk broadcast)  | ~30 сообщений / сек                  |

### Paid Broadcasts — не применимо к нашей архитектуре

Telegram-тариф Paid Broadcasts снимает лимит до 1000 msgs/sec, но **для нашего use-case не нужен**:

- **Лимиты считаются per-bot**. У нас каждый клиент держит свой бот → лимит 30/sec у каждого свой и не пересекается с другими клиентами.
- Чтобы клиент включил Paid Broadcasts на **своём** боте, ему нужно **100k Stars + 100k MAU** на этом боте — нереально для типичного бренда.
- Для SMM-сценария (плановые посты 1-5/день в канал) 30/sec **с большим запасом**.

Детали тарифа на случай, если когда-то понадобится:
- Потолок: **1000 сообщений / сек** per-bot.
- Цена: **0.1 Telegram Stars** за сообщение сверх бесплатных 30/сек.
- Требования: на балансе бота **>= 100 000 Stars** и **>= 100 000 monthly active users**.
- Платится со Stars-баланса самого бота, только за успешно доставленные.

## 8. Особенности

- **App review / sandbox отсутствуют** — бот сразу боевой, никаких ревью платформы.
- **Двухэтапное ручное подключение клиентом в Telegram UI** — клиент сам (1) создаёт свой бот в @BotFather, копирует токен и вставляет в наш интерфейс; (2) добавляет своего бота админом в свой канал с правом `can_post_messages`. Оба действия — техническая работа в Telegram-приложении, не в нашем UI. Для не-программиста это нетривиально, нужна подробная пошаговая инструкция (текст + скриншоты или GIF).
- **Длина подписи 1024 ≠ длине сообщения 4096** — для длинных постов с медиа: отдельно медиа без подписи, отдельно `sendMessage` с текстом.

## 9. Что учесть при запуске интеграции

- **UX подключения канала**: клиент сам (1) создаёт бот через @BotFather, копирует токен и вставляет к нам; (2) добавляет своего бота админом в свой канал с правом `can_post_messages`. Оба шага — действия в Telegram-приложении, требуют пошаговой инструкции в нашем UI.

## Карточка для сводки

| Параметр                                 | Значение                                              |
|------------------------------------------|-------------------------------------------------------|
| Сложность интеграции                     | low                                                   |
| Тип авторизации                          | bot token (статический)                               |
| Требуется app review                     | нет                                                   |
| Поддержка отложенного постинга в API     | нет (хранить очередь у нас)                           |
| Поддержка карусели/альбома               | да (`sendMediaGroup`, 2-10 элементов)                 |
| Лимит длины текста                       | 4096 символов (подпись медиа 1024)                    |
| Макс размер медиа                        | 50 MB multipart / 20 MB URL / 2000 MB self-hosted     |
| Rate limit                               | 30 rps **на каждый бот клиента** (per-bot), 20/мин на группу, 1/сек на чат |
| Платный тариф для повышения лимитов      | существует (Paid Broadcasts), скорее всего неприменим к нашему use-case |
| Подходит для «draft → review → publish»  | да — очередь и approval полностью у нас               |
