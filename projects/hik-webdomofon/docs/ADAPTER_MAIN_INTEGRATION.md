# План интеграции HikvisionAdapter в `hik-talk/app/main.py`

> Статус: **план, НЕ реализован**. Изменения в `main.py` в этот документ не вносятся.
> Предусловие: latent bug исправлен (guard в `start_voice_com` / `send_voice_data`),
> 48 unit-тестов адаптера зелёные, полный suite: 146 passed / 5 skipped.

## 1. Цель

Заменить raw ctypes-код (`cdll.LoadLibrary` + `sdk.NET_DVR_*`) на `HikvisionAdapter`
без изменения поведения Voice Talk, WebSocket-эндпоинтов, формата аудио и внешнего API.

## 2. Точки замены в `main.py` (по строкам текущего файла)

| Зона | Строки | Сейчас | Станет |
|---|---|---|---|
| A. Импорты | 3–4 | `from ctypes import *`, `from HCNetSDK import *` | `from hik_adapter import get_adapter, HikAdapterError, NET_DVR_DVRVOICEOPENED` |
| B. Путь SDK | 129 | `from HCNetSDK import netsdkdllpath` | удалить (путь резолвит адаптер) |
| C. Загрузка + Init | 131–133 | `sdk = cdll.LoadLibrary(netsdkdllpath)`; `sdk.NET_DVR_Init()` | `adapter = get_adapter()`; `adapter.initialize()` |
| D. Прототипы | 135–161 | 6× `.argtypes/.restype` | удалить (внутри `_apply_prototypes()`) |
| E. Логин | 168–189 | `login = NET_DVR_USER_LOGIN_INFO()` … `USER_ID = sdk.NET_DVR_Login_V40(...)`; `if USER_ID < 0: raise RuntimeError(...)` | `USER_ID = adapter.login()` (внутри retry + `LoginError`; при неудаче `LoginError` унаследован от `RuntimeError` → существующий fail-fast сохранён) |
| F. Константа 29 | 195 | `NET_DVR_DVRVOICEOPENED = 29` | удалить (импортируется из `hik_adapter`) |
| G. `/talk` WebSocket | 233–300 | `sdk.NET_DVR_StartVoiceCom_MR_V30(USER_ID, VOICE_CHANNEL, None, None)`; `sdk.NET_DVR_VoiceComSendData(...)`; `sdk.NET_DVR_StopVoiceCom(handle)`; `sdk.NET_DVR_GetLastError()` | `adapter.start_voice_com(VOICE_CHANNEL)`; `adapter.send_voice_data(handle, buf, WS_ULAW_FRAME_BYTES)`; `adapter.stop_voice_com(handle)`; `adapter.get_last_error()` |
| H. `VoiceSession._open_channel` | 433–440 | `sdk.NET_DVR_StartVoiceCom_MR_V30(USER_ID, VOICE_CHANNEL, None, None)` | `adapter.start_voice_com(VOICE_CHANNEL)` |
| I. `VoiceSession._close_channel` | 443–446 | `sdk.NET_DVR_StopVoiceCom(self.handle)` | `adapter.stop_voice_com(self.handle)` |
| J. `VoiceSession._pump_frames` | 406–416 | `sdk.NET_DVR_VoiceComSendData(...)`; `sdk.NET_DVR_GetLastError()`; `err == NET_DVR_DVRVOICEOPENED` | `adapter.send_voice_data(...)` в `try/except VoiceComError as e:`; `e.code == NET_DVR_DVRVOICEOPENED` |
| K. `VoiceSession.run` (retry 29) | 378–388 | `sdk.NET_DVR_GetLastError()` → `err == NET_DVR_DVRVOICEOPENED` | `except VoiceComError as e:` → `e.code == NET_DVR_DVRVOICEOPENED` |
| L. `_notify_error` | 458 | dict с ключом `NET_DVR_DVRVOICEOPENED` | без изменений (константа импортирована из адаптера) |

## 3. Семантический маппинг ошибок

| Raw-код сейчас | Адаптер | Эквивалент |
|---|---|---|
| `StartVoiceCom` вернул `-1`/`None` → `GetLastError()` | `VoiceComError.code` | `err.code` |
| `VoiceComSendData` вернул `False` → `GetLastError()` | `VoiceComError.code` | `err.code` |
| `err == 29` (канал занят) | `err.code == NET_DVR_DVRVOICEOPENED` (== 29) | идентично |
| `Login_V40 < 0` → `RuntimeError(f"Login failed: {GetLastError()}")` | `LoginError` (наследник `RuntimeError`), `.code` | `str(e)` содержит camera; при желании `.code` |

## 4. Ключевые моменты эквивалентности (не сломать)

1. **Логин остаётся однократным при импорте модуля**: `get_adapter()` — singleton;
   `adapter.login()` идемпотентен (кэширует `_user_id`).
2. **`USER_ID` остаётся доступен** как значение, возвращённое `adapter.login()` —
   используется в `/talk` (G) и `VoiceSession` (H).
3. **`sdk` больше не существует** как имя в модуле — проверить, что нигде больше
   в `main.py` (и в `app/` импортирующих его модулях) нет ссылок на `sdk.`.
4. **Аудио-формат не трогаем**: `send_voice_data(handle, buf, WS_ULAW_FRAME_BYTES)`
   — байты кадра и размер передаются как раньше (мнемоника «160 байт G711U»).
5. **Backoff при 29** сохраняется: `WS_RETRY_DELAYS` и `_notify_error(reconnecting=True)`
   — только источник `err.code` меняется с `sdk.NET_DVR_GetLastError()` на `e.code`.
6. **`import time` внутри login-ретраев адаптера** — не конфликтует с `time.monotonic()`
   в `_pump_frames` (разные модули/атрибуты).

## 5. Шаги внедрения (когда будет одобрено)

1. Заменить блок A–F (импорты, загрузка, логин) одним патчем.
2. Заменить G (`/talk`) — самый простой патч, проверить PTT-консоль вручную.
3. Заменить H–K (`VoiceSession`) — прогнать `tests/test_voice_session.py` + `test_fastapi_routes.py`.
4. Прогнать полный suite: **ожидание 146 passed / 5 skipped** (без новых тестов).
5. Вручную: `python app/main.py` с реальной камерой — PTT, `/ws/audio`, занятость (29), backoff.

## 6. Что НЕ делать в этом изменении

- Не менять `talk_server.py`, `tools/voice_test.py` (отдельные задачи).
- Не удалять пакет `HCNetSDK` (нужен тестам `test_sdk_*`).
- Не менять Docker / `VOICE_CHANNEL` / `WS_*` константы / WebSocket-протокол.
- Не выносить `USER_ID` в глобальный адаптер, если голосовой поток ещё жив.

## 7. Риски и митигация

| Риск | Митигация |
|---|---|
| Где-то остался `sdk.` → `AttributeError` | `grep -rn "sdk\." hik-talk/app/` после патча — 0 совпадений |
| `LoginError` не перехвачен на старте | `LoginError` наследует `RuntimeError` → `raise` на уровне модуля сохраняет fail-fast |
| Двойной Init/Login при переимпорте `main` в тестах | `initialize()`/`login()` идемпотентны; conftest мокает `cdll.LoadLibrary` |
| Изменение поведения ошибки 29 | Тесты `test_voice_session.py` (21 шт.) покрывают retry/backoff — должны остаться зелёными |
