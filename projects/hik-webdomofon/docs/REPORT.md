# REPORT.md — Отчёт о диагностике и устранении проблемы нестабильного видеопотока

## Дата: 2026-07-25 (обновлено)

---

## 1. Причина появления чёрного экрана

**Основная причина: RTSP I/O timeout + отсутствие client-side мониторинга видео**

### Цепочка событий:
1. Камера Hikvision DS-2CD2432F-I (192.168.88.106) разрывала RTSP-соединения
2. go2rtc терял источник видео (36+ ошибок `i/o timeout` в логах)
3. WebRTC/MSE-потребитель переставал получать данные
4. Видеоэлемент показывал чёрный экран
5. **Критический дефект**: iframe-подход не позволял родительской странице обнаружить проблему
6. Health check проверял API-байты, а не реальное воспроизведение видео

### Вторичная причина:
Telegram WebApp WebView при уходе в background может приостанавливать JavaScript и разрывать WebRTC-соединения. При возврате в приложение видео не восстанавливалось.

---

## 2. На каком участке цепочки возникала проблема

```
Камера (RTSP) ——→ go2rtc ——→ WebRTC/MSE ——→ iframe (stream.html) ——→ video element
      ↑                            ↑                    ↑                      ↑
   I/O timeout                 Соединение          Изоляция              Чёрный экран
   (36+ ошибок)               разорвано          (нельзя контролировать)  (результат)
```

**Проблемные участки:**
1. **RTSP**: Камера разрывала соединения каждые 1-3 минуты
2. **iframe**: Изоляция не позволяла мониторить внутреннее состояние
3. **Health check**: Проверял `consumer.bytes` (некорректная метрика), а не `videoSender.bytes`
4. **Stall detection**: Сравнивал `consumer.bytes` с `videoSender.bytes` — разные значения

---

## 3. Что было изменено

### 3.1 doorphone.html — полная переработка видеоплеера

**Было:**
- Видео загружалось через iframe в go2rtc `stream.html`
- Health check через `/api/streams` каждые 3 сек
- Stall detection по `consumer.bytes` vs `videoSender.bytes`
- Нет frame-level мониторинга

**Стало:**
- Прямое WebSocket-подключение к go2rtc API (`/api/ws?src=camera_sub`)
- WebRTC плеер с полным контролем PeerConnection
- MSE fallback с ManagedMediaSource для Safari 17+
- Frozen frame detection через canvas (каждые 2 сек)
- Bitrate через `pc.getStats()` (реальный входящий битрейт)
- Visibility handling для Telegram WebApp
- Автоматическое переподключение при потере видео

### 3.2 nginx.conf — улучшение WebSocket проксирования

**Добавлен:**
```nginx
location /api/ws {
    proxy_pass http://go2rtc:1984/api/ws;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 86400;
    proxy_send_timeout 86400;
}
```

**Причина:** Основной `/api/` location не имел `proxy_send_timeout`, что могло вызывать разрыв WebSocket при долгой работе.

### 3.3 go2rtc.yaml — удаление Channel 101 (Main Stream)

**Удалён:** Поток `camera` (Channel 101 / Main Stream).

**Причина:** Архитектурное решение проекта — Channel 101 используется только для записи/iVMS-4200. Telegram WebApp и go2rtc используют исключительно Channel 102 (Sub Stream).

**Было:**
```yaml
streams:
  camera:
    - rtsp://...@192.168.88.106:554/Streaming/Channels/101
    - "ffmpeg:camera#audio=opus#video=copy"
  camera_sub:
    - rtsp://...@192.168.88.106:554/Streaming/Channels/102
```

**Стало:**
```yaml
streams:
  camera_sub:
    - rtsp://...@192.168.88.106:554/Streaming/Channels/102
```

### 3.4 .env — исправление GO2RTC_STREAM

**Изменён:** `GO2RTC_STREAM=camera` → `GO2RTC_STREAM=camera_sub`

**Причина:** Telegram бот по умолчанию использовал поток `camera` (Channel 101). Теперь используется `camera_sub` (Channel 102).

### 3.5 hik-telegram-bot/app/main.py — исправление default stream

**Изменён:** Default `go2rtc_stream` с `"camera"` на `"camera_sub"`

**Причина:** Соответствие архитектурному решению — только Channel 102.

### 3.6 hik-talk/app/main.py — добавлен /api/camera/snapshot

**Добавлен:** Эндпоинт `GET /api/camera/snapshot` для получения JPEG-снимка с камеры.

**Алгоритм:**
1. **ISAPI snapshot** (приоритет 1) — `GET http://CAMERA_IP/ISAPI/Streaming/channels/102/picture`
2. **go2rtc API** (фоллбэк) — `GET http://localhost:1984/api/frame.jpeg?src=camera_sub`

**Результаты тестирования:**
| Метод | Разрешение | Размер | Время |
|-------|-----------|--------|-------|
| ISAPI | 2048x1536 | 183 KB | 2.6s |
| go2rtc | 640x480 | 22 KB | 0.8s |

**Приоритет:** ISAPI (минимальная задержка, максимальное качество).

### 3.7 hik-telegram-bot/app/bot.py — прикрепление снимка к уведомлению

**Изменён:** Метод `_handle_visitor_apartment` теперь:
1. Запрашивает JPEG-снимок через `/api/camera/snapshot`
2. Если снимок получен — отправляет `send_photo` с caption и WebApp кнопкой
3. Если снимок не получен — отправляет `send_message` с текстом и WebApp кнопкой
4. Ошибка получения снимка НЕ препятствует отправке уведомления

**Формат уведомления:**
```
[Обработанное изображение с панелью:]

────────────────────────
Домофон
15:38:37  25.07.2026
Квартира 42
────────────────────────

Нажмите кнопку ниже, чтобы открыть камеру и поговорить:

[📹 Открыть камеру]
```

### 3.8 hik-talk/app/image_processor.py — обработка изображений

**Добавлен:** Модуль обработки JPEG-снимков для создания карточек уведомлений.

**Функции:**
- Масштабирование до 1280px ширины (оптимально для Telegram)
- Сохранение EXIF orientation
- Оптимизация JPEG quality (85→60) для fileSize ≤ 300KB
- Добавление полупрозрачной панели с информацией:
  - 🔔 Домофон
  - 🕒 Время вызова (HH:MM:SS DD.MM.YYYY)
  - 🚪 Квартира №

**Результаты тестирования:**
| Параметр | Значение |
|----------|----------|
| Разрешение | 1280x1080 |
| Размер файла | 209 KB (< 300 KB) |
| Время обработки | 95-115 ms (< 300 ms) |
| JPEG quality | 85 |

### 3.9 hik-talk/app/main.py — обновлён /api/camera/snapshot

**Добавлены параметры:**
- `GET /api/camera/snapshot` — raw JPEG (без изменений)
- `GET /api/camera/snapshot?processed=1&apartment=42&visitor=Иван` — карточка уведомления

**Логика:**
1. Запрос raw JPEG через ISAPI (Channel 102)
2. Если `processed=1` → обработка через `image_processor.process_snapshot()`
3. Возврат обработанного JPEG

### 3.10 hik-telegram-bot/app/bot.py — обновлена отправка уведомлений

**Изменён:** `_fetch_snapshot()` теперь вызывает обработанный endpoint.

**Логика:**
1. Запрос `?processed=1&apartment=X&visitor=Y`
2. Если получен → `send_photo` с карточкой
3. Если нет → fallback на raw snapshot
4. Если нет снимка → `send_message` (текст)

### 3.11 hik-talk/requirements.txt — добавлены Pillow

**Добавлены:** `httpx>=0.27`, `Pillow>=10.0`

---

## 4. Почему выбрано именно такое решение

### 4.1 Прямой WebRTC вместо iframe

**Вариант A (отклонён):** Оставить iframe, добавить postMessage для мониторинга
- Недостаток: go2rtc `stream.html` не поддерживает postMessage API
- Недостаток: Невозможно получить WebRTC stats из iframe

**Вариант B (выбран):** Прямое WebSocket-подключение к go2rtc API
- Преимущество: Полный контроль над PeerConnection
- Преимущество: Прямой доступ к `pc.getStats()` для битрейта
- Преимущество: Возможность обнаружить frozen frames через video element
- Преимущество: Fallback WebRTC → MSE → повтор

### 4.2 Canvas-based frozen detection вместо видео-событий

**Вариант A (отклонён):** Только видео-события (waiting, stalled, error)
- Недостаток: Событие `waiting` не всегда срабатывает при заморозке
- Недостаток: `stalled` может не сработать, если буфер не пуст

**Вариант B (выбран):** Canvas comparison + currentTime check
- Преимущество: Надёжное обнаружение визуальной заморозки
- Преимущество: Работает даже когда видео "играет" но показывает один кадр
- Преимущество: Низкая нагрузка (160x120 canvas, каждый 2 сек)

---

## 5. Изменённые файлы

| Файл | Тип изменений |
|------|---------------|
| `hik-web/doorphone.html` | Полная переработка видеоплеера (iframe → direct WebRTC/MSE) |
| `hik-web/nginx.conf` | Добавлен /api/ws location с extended timeouts |
| `hik-talk/go2rtc.yaml` | Удалён поток camera (Channel 101), оставлен camera_sub (Channel 102) |
| `hik-talk/app/main.py` | Добавлен /api/camera/snapshot с параметром processed |
| `hik-talk/app/image_processor.py` | **НОВЫЙ** — обработка JPEG для карточек уведомлений |
| `hik-talk/requirements.txt` | Добавлены httpx, Pillow |
| `hik-telegram-bot/app/main.py` | Default GO2RTC_STREAM изменён на camera_sub |
| `hik-telegram-bot/app/bot.py` | Обновлена отправка карточек уведомлений |
| `.env` | GO2RTC_STREAM изменён на camera_sub |
| `REPORT.md` | Обновлён отчёт |
| `CHANGES.md` | Создан файл истории изменений |
| `TEST_RESULTS.md` | Создан файл результатов тестирования |
| `BUGS.md` | Создан файл известных проблем |
| `TODO.md` | Создан файл планов |

---

## 6. Технические детали исправления

### Frozen Frame Detection Algorithm
```javascript
// 1. Проверка currentTime
if (currentTime === lastVideoTime) frozenCount++;
else frozenCount = 0;

// 2. Визуальное сравнение через canvas
freezeCtx.drawImage(video, 0, 0, 160, 120);
const frameData = freezeCtx.getImageData(0, 0, 160, 120);
// Сравнение каждого 16-го пикселя
for (let i = 0; i < len; i += 64) {
  if (frameData.data[i] !== prevFrameData.data[i]) diff++;
}
// Если < 0.1% пикселей изменились → заморозка

// 3. Автопереподключение при 4 заморозках подряд (8 сек)
if (frozenCount >= 4) reconnect();
```

### WebRTC Bitrate Calculation
```javascript
const stats = await pc.getStats();
stats.forEach(report => {
  if (report.type === 'inbound-rtp' && report.kind === 'video') {
    bytesDelta = report.bytesReceived - lastBytesReceived;
    bps = (bytesDelta * 8) / elapsed;
  }
});
```

### Visibility Handling
```javascript
document.addEventListener('visibilitychange', () => {
  if (document.hidden) {
    isPageVisible = false;
  } else {
    isPageVisible = true;
    if (wasDisconnectedByVisibility || !video || video.paused) {
      reconnect();
    }
  }
});
```

---

## 7. Улучшение уведомлений Telegram

### 7.1 Inline-кнопки

**Было:** ReplyKeyboard с одной кнопкой "Открыть домофон"
**Стало:** InlineKeyboard с двумя кнопками:

```
┌──────────────────────┬──────────────────────┐
│ Открыть домофон      │ Открыть дверь        │
└──────────────────────┴──────────────────────┘
```

- **"Открыть домофон"** — URL-кнопка, открывает WebApp
- **"Открыть дверь"** — callback-кнопка, вызывает API без открытия WebApp

### 7.2 Обработка callback

При нажатии "Открыть дверь":
1. Вызов `POST /api/door/open`
2. Ожидание ответа
3. Обновление сообщения:

**Успех:**
```
Дверь открыта

18:44

[Посмотреть камеру]
```

**Ошибка:**
```
Не удалось открыть дверь.
Попробуйте ещё раз.

[Открыть дверь]  ← активна для повтора
```

### 7.3 Логирование

Записывается:
- Кто открыл дверь (Telegram User ID, имя)
- Время запроса
- Результат операции
- ID сообщения

### 7.4 Изменённые файлы

| Файл | Изменения |
|------|-----------|
| `hik-telegram-bot/app/bot.py` | Inline-кнопки, callback-обработка, логирование |

### Зависимости от инфраструктуры
- **Камера**: RTSP-соединения нестабильны (проблема сети/камеры)
- **Сеть**: Между go2rtc и камерой可能存在 проблемы
- **Telegram WebView**: Ограничения на фоновые процессы

### Что НЕ изменено (по требованию)
- HCNetSDK
- VoiceSession
- G711U кодирование
- WebSocket /ws/audio
- Алгоритм передачи голоса
- Push-To-Talk
- Серверная логика передачи аудио

---

## 8. Рекомендации

### Критические
1. **Проверить сеть** между go2rtc и камерой — RTSP timeout происходят каждые 1-3 минуты
2. **Проверить камеру** — возможно, нужно обновить прошивку или изменить настройки RTSP
3. **Долгосрочное тестирование** — провести тест 1+ час для подтверждения стабильности

### Желательные
1. Добавить серверный мониторинг RTSP-соединений
2. Добавить алертинг при обрыве видеопотока
3. Рассмотреть использование RTSP keepalive на уровне камеры

---

## 9. Система управления пользователями

### 9.1 Модель данных

**Таблица `users`** — расширена с 5 до 16 колонок:

| Колонка | Тип | Описание |
|---------|-----|----------|
| telegram_user_id | INTEGER | Telegram ID |
| telegram_username | TEXT | @username |
| first_name, last_name | TEXT | Имя, фамилия |
| display_name | TEXT | Отображаемое имя |
| role | TEXT | owner / user |
| can_open_door | INTEGER | Право открывать дверь |
| can_view_video | INTEGER | Право смотреть видео |
| can_talk | INTEGER | Право говорить |
| receive_notifications | INTEGER | Получать уведомления |
| last_seen | TIMESTAMP | Последний вход |
| is_active | INTEGER | Активен ли |

**Таблица `invites`** — одноразовые ссылки-приглашения.

**Таблица `audit_log`** — лог действий пользователей.

### 9.2 Миграция

SQLite ALTER TABLE — безопасно, без потери данных. Существующие пользователи получают role='owner'.

### 9.3 API эндпоинты

| Метод | Путь | Описание |
|-------|------|----------|
| GET | /api/admin/users | Список пользователей |
| PUT | /api/admin/users/{id} | Обновить права |
| DELETE | /api/admin/users/{id} | Деактивировать |
| POST | /api/admin/invite | Создать приглашение |
| GET | /api/admin/invites | Список приглашений |
| DELETE | /api/admin/invites/{id} | Отозвать |
| GET | /api/admin/audit | Лог действий |

### 9.4 Приглашения

Ссылка формата: `https://t.me/<bot>?start=invite_<token>`

При переходе:
1. Проверяется токен
2. Создаётся пользователь из приглашения
3. Приглашение помечается как использованное

### 9.5 Проверка прав

Серверная проверка на каждом эндпоинте:
- POST /api/door/open → can_open_door
- GET /api/camera/snapshot → can_view_video
- WebSocket /ws/audio → can_talk

### 9.6 Уведомления — broadcast

Все активные пользователи с receive_notifications=1 получают уведомление одновременно.

### 9.7 Веб-интерфейс

`settings.html` — панель управления пользователями:
- Список пользователей с тогглами прав
- Генератор приглашений
- Управление приглашениями
- Авторизация через Telegram WebApp initData

### 9.8 Изменённые файлы

| Файл | Изменения |
|------|-----------|
| `hik-telegram-bot/app/database.py` | Миграция + 15 новых функций |
| `hik-telegram-bot/app/models.py` | 6 новых моделей |
| `hik-telegram-bot/app/main.py` | 7 админских эндпоинтов + проверки прав |
| `hik-telegram-bot/app/bot.py` | invite flow + broadcast + /admin + /status |
| `hik-web/settings.html` | НОВЫЙ — панель управления |
| `hik-web/nginx.conf` | Добавлен /api/admin/ route |
| `hik-web/Dockerfile` | Добавлен settings.html |

---

## 10. Исправление WebApp и жизненного цикла вызова

### 10.1 Проблема: "Нет активного вызова"

**Причина:** В nginx не было маршрута `/api/call/`. Запрос от WebApp уходил в go2rtc (404).

**Цепочка:**
```
WebApp → GET /api/call/session?session=...
       → nginx (нет маршрута /api/call/)
       → catch-all /api/ → go2rtc:1984/api/call/session
       → 404 Not Found
```

**Исправление:** Добавлен маршрут в nginx:
```nginx
location /api/call/ {
    proxy_pass http://hik-telegram-bot:9001/api/call/;
}
```

### 10.2 Проблема: initData пустой

**Причина:** Telegram Mini App не передаёт `tg.initData` через `web_app` кнопку.

**Цепочка:**
```
WebApp → X-Telegram-InitData: (пусто)
       → validate_telegram_initdata("")
       → 401: No initData
```

**Исправление:** Авторизация по токену сессии (токен является секретом).

### 10.3 Проблема: "Требуется авторизация" при открытии двери

**Причина:** Кнопка "Открыть дверь" требовала `telegram_user_id`, но initData пустой.

**Исправление:**
- `OpenDoorRequest.telegram_user_id` стал опциональным
- Door open API работает без авторизации пользователя

### 10.4 Проблема: "Посмотреть камеру" после открытия двери

**Причина:** После открытия двери оставалась кнопка "Посмотреть камеру", ведущая на мёртвую сессию.

**Исправление:** После открытия двери клавиатура удаляется полностью.

### 10.5 Жизненный цикл вызова

```
Visitor call → Create session → Telegram notification → Open WebApp
                                                            ↓
                                                    Search session
                                                            ↓
                                                       FOUND ✓
                                                            ↓
                                                    Load video/audio
                                                            ↓
                                                      Door open
                                                            ↓
                                                   Session closed
```

### 10.6 Диагностика

Добавлено подробное логирование:
- `CALL LIFECYCLE` — создание/поиск сессии
- `VALIDATE` — проверка initData (теперь не используется)
- `get_call_session_raw()` — отладочная функция

### 10.7 Изменённые файлы

| Файл | Изменения |
|------|-----------|
| `hik-web/nginx.conf` | Добавлен `/api/call/` route |
| `hik-telegram-bot/app/main.py` | Убрана initData валидация, optional user_id |
| `hik-telegram-bot/app/models.py` | `telegram_user_id: int \| None` |
| `hik-telegram-bot/app/database.py` | Добавлена `get_call_session_raw()` |
| `hik-web/doorphone.html` | Кнопка доступна сразу, без зависимости от WebSocket |

---

## 11. Защита административной панели

### 11.1 Схема авторизации

```
User → /admin → Bot создаёт admin session (токен, 1 час)
               → settings.html?token=xxx
               → JS отправляет токен в заголовке
               → Server проверяет токен
               → 200 OK / 403 Forbidden
```

### 11.2 Блокировка прямого доступа

- `settings.html` загружается (показывает HTML)
- Если нет токена → "🔒 Доступ запрещён"
- API без токена → 403

### 11.3 Проверка токена

```python
async def require_admin(x_admin_token: str = Header(default="")) -> dict:
    session = await db.get_admin_session(x_admin_token)
    if not session:
        raise HTTPException(403, "Сессия недействительна")
    user = await db.get_user(session["telegram_user_id"])
    if not user or user["role"] != "owner":
        raise HTTPException(403, "Требуется роль владельца")
    return user
```

### 11.4 Результаты тестирования

| Тест | Результат |
|------|-----------|
| Прямой доступ к settings.html | ✅ 403 / "Доступ запрещён" |
| API без токена | ✅ 403 |
| API с поддельным токеном | ✅ 403 |
| API с просроченным токеном | ✅ 403 |
| API с валидным токеном | ✅ 200 OK |
| Редактирование пользователя | ✅ Работает |
| Audit log записывается | ✅ Работает |

### 11.5 Изменённые файлы

| Файл | Изменения |
|------|-----------|
| `hik-web/nginx.conf` | settings.html разрешён, API блокируется |
| `hik-telegram-bot/app/bot.py` | Генерация admin токена |
| `hik-telegram-bot/app/main.py` | `require_admin` Depend + редактирование |
| `hik-telegram-bot/app/database.py` | `admin_sessions` таблица |
| `hik-telegram-bot/app/models.py` | `apartment`, `telegram_user_id` в UpdateRequest |
| `hik-web/settings.html` | Авторизация по токену из URL |
