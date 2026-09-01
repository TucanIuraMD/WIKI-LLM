# План изолированного HikvisionAdapter

> На основе аудита `docs/AUDIT_2026-08-19.md` (минимальный граф зависимостей HCNetSDK).
> Статус: **проектирование**. Код адаптера создан, но **не подключён к production**.

## 1. Цели и ограничения

- **Изолированность**: адаптер не импортирует пакет `HCNetSDK` (ни `HCNetSDK.py`, ни `SDK_*`), зависит только от stdlib (`ctypes`, `os`, `logging`) и нативной `libhcnetsdk.so`.
- **Не удалять существующий SDK** — пакет `hik-talk/HCNetSDK/` остаётся нетронутым (нужен тестам и как источник точных layout структур).
- **Не менять поведение Voice Talk** — сохраняются 6 вызовов, их сигнатуры, семантика ошибки 29 (`NET_DVR_DVRVOICEOPENED`), «одна сессия на процесс», retry-логика.
- **Не менять API Backend** — `app/main.py`, `talk_server.py`, `tools/voice_test.py` не модифицируются на этом этапе.

## 2. План файлов

| Файл | Действие | Статус |
|---|---|---|
| `hik-talk/app/hik_adapter.py` | **Новый**: изолированный адаптер | ✅ создан |
| `docs/ADAPTER_PLAN.md` | **Новый**: этот документ | ✅ создан |
| `docs/AUDIT_2026-08-19.md` | Без изменений (база для дизайна) | — |
| `hik-talk/HCNetSDK/*` | Без изменений (существующий SDK) | — |
| `hik-talk/app/main.py` | **НЕ трогать** (подключение — отдельный этап) | — |
| `hik-talk/talk_server.py` | **НЕ трогать** (legacy) | — |
| `tools/voice_test.py` | **НЕ трогать** (может быть переведён позже) | — |
| `hik-talk/tests/test_hik_adapter.py` | **Планируется** (мок `cdll.LoadLibrary` через conftest) | todo |

## 3. Интерфейс адаптера

### 3.1 Публичные константы и исключения

```python
NET_DVR_DVRVOICEOPENED = 29      # канал голосовой связи занят

class HikAdapterError(RuntimeError): ...   # базовое
class SdkLoadError(HikAdapterError): ...   # .so не найден/не загружен
class LoginError(HikAdapterError): ...     # .code — SDK error code
class VoiceComError(HikAdapterError): ...  # .code — SDK error code
```

### 3.2 Конфигурация (env, совместимо с текущим `.env`)

| Переменная | Назначение |
|---|---|
| `HIK_SDK_PATH` | путь к `libhcnetsdk.so` (новый, приоритетный) |
| `CAMERA_IP`, `CAMERA_PORT`, `CAMERA_USER`, `CAMERA_PASSWORD` | параметры логина |

### 3.3 Класс `HikvisionAdapter`

```python
class HikvisionAdapter:
    def __init__(self, sdk_path=None, ip=None, port=8000,
                 username=None, password=None, login_retries=3): ...

    # lifecycle
    def initialize(self) -> None            # load .so + NET_DVR_Init (идемпотентно)
    def login(self) -> int                  # NET_DVR_Login_V40 → user_id (одна сессия)
    def logout(self) -> bool                # NET_DVR_Logout (идемпотентно)
    def cleanup(self) -> None               # NET_DVR_Cleanup (идемпотентно)

    # voice talk
    def start_voice_com(self, channel=0) -> int    # StartVoiceCom_MR_V30 → handle
    def send_voice_data(self, handle, data, size=160) -> None  # VoiceComSendData
    def stop_voice_com(self, handle) -> bool       # StopVoiceCom (идемпотентно)
    def get_last_error(self) -> int                # NET_DVR_GetLastError

def get_adapter() -> HikvisionAdapter:  # process-wide singleton
```

### 3.4 Маппинг на production-семантику

| Текущий код (app/main.py) | Адаптер | Поведение сохранено |
|---|---|---|
| `sdk = cdll.LoadLibrary(netsdkdllpath)` | `initialize()` + `resolve_sdk_path()` | ✅ поиск пути: env → пакет-relative → CWD |
| `sdk.NET_DVR_Init()` | `initialize()` | ✅ один раз |
| `NET_DVR_Login_V40(byref(login), byref(dev))` | `login()` | ✅ user_id ≥ 0; `LoginError` при fail (аналог `RuntimeError`) |
| `StartVoiceCom_MR_V30(USER_ID, VOICE_CHANNEL, None, None)` | `start_voice_com(channel)` | ✅ handle != -1 |
| `VoiceComSendData(handle, frame, 160)`; при `not ok` → `GetLastError()` == 29 | `send_voice_data(handle, data)` → `VoiceComError(code=29)` | ✅ вызывающий сравнивает `err.code == NET_DVR_DVRVOICEOPENED` |
| `StopVoiceCom(handle)` | `stop_voice_com(handle)` | ✅ idempotent, handle == -1 → True |
| `NET_DVR_DVRVOICEOPENED = 29` (локально в main.py) | константа модуля адаптера | ✅ |
| логин при импорте (одна сессия) | `get_adapter()` singleton | ✅ |

## 4. Что инкапсулировано внутри адаптера

- **ctypes-структуры** (точные C-layout, скопированы из `SDK_Struct.py`):
  - `NET_DVR_DEVICEINFO_V30` (вложена в V40)
  - `NET_DVR_DEVICEINFO_V40`
  - `_LOGIN_CALLBACK` (аналог `fLoginResultCallBack`)
  - `NET_DVR_USER_LOGIN_INFO`
- **argtypes/restype** для всех 6 функций (+ `NET_DVR_Logout`, `NET_DVR_Cleanup`)
- **Резолвинг пути** к `libhcnetsdk.so` (`HIK_SDK_PATH` → relative → CWD)
- **Семантика ошибок**: `LoginError` / `VoiceComError` с `.code`, сравнение с 29

## 5. Что НЕ входит в адаптер (осознанно)

- `NetClient` (класс-обёртка) — не используется production
- Видео/PlaySDK (`RealPlay_V40`, `PlayM4_*`) — не используется
- PTZ, алермы, XML-конфиг, ID-карты — не используется
- `SDK_Enum.py` / `SDK_Callback.py` — не нужны (`fLoginResultCallBack` продублирован в адаптере)

## 6. Порядок подключения (отдельный этап, НЕ выполнен)

1. Тесты адаптера с моком `cdll.LoadLibrary` (паттерн conftest.py уже есть).
2. Замена в `app/main.py`: `from HCNetSDK import *` + raw `sdk.*` → `from hik_adapter import get_adapter`.
3. Прогон `pytest tests/` — 98 тестов должны остаться зелёными.
4. Ручная проверка Voice Talk на реальной камере (PTT, занятость канала error 29).
5. Опционально: перевод `tools/voice_test.py` на адаптер (убрать сломанный хардкод `/root/hik-venv/...`).

## 7. Тестовый план для адаптера (todo)

| Тест | Суть |
|---|---|
| `test_resolve_sdk_path` | env → relative → None |
| `test_initialize_mock` | мок `LoadLibrary`; прототипы выставлены; Init вызван |
| `test_login_success` | user_id ≥ 0; повторный `login()` не вызывает повторно `Login_V40` |
| `test_login_failure` | user_id < 0 → `LoginError` c `.code` |
| `test_start_voice_com` | handle; handle == -1 → `VoiceComError` |
| `test_send_voice_data` | ok=False, code=29 → `VoiceComError(code=29)` |
| `test_stop_voice_com` | idempotent; handle=-1 → True |
| `test_cleanup` | Cleanup вызван; повторный вызов безопасен |
