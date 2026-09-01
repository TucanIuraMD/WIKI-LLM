# Module Overview

This document unifies the information from `docs/MODULE_CALLS.md` and `docs/MODULE_DEPENDENCIES.md` to give a single reference of internal imports and external dependencies.

---

## 1. Import Map (from docs/MODULE_CALLS.md)

### hik‑talk package (`hik‑talk/app`)
```
app/main.py
├─ imports FastAPI, WebSocket, uvicorn, asyncio, threading, queue, json, logging, time, os
├─ imports HCNetSDK (package under `hik‑talk/HCNetSDK`)
│   └─ HCNetSDK/__init__.py re‑exports symbols from HCNetSDK.py, SDK_Enum.py, SDK_Struct.py, SDK_Callback.py
├─ imports image_processor (local module `hik‑talk/app/image_processor.py`)
└─ imports httpx (for snapshot fetching)
```

### hik‑telegram‑bot package (`hik‑telegram‑bot/app`)
```
main.py
├─ imports FastAPI, HTTPException, Header, Query, Depends, CORSMiddleware
├─ imports .database as db (same package)
├─ imports .models (pydantic models)
├─ imports .bot (build_bot function)
└─ uses httpx for HTTP calls to hik‑talk and Telegram API
```

`bot.py`
```
├─ imports .database as db
├─ imports telegram, telegram.ext (python‑telegram‑bot library)
├─ imports httpx for snapshot fetching and bot‑to‑API calls
└─ uses OS env variables for configuration
```

### HCNetSDK package (`hik‑talk/HCNetSDK`)
```
HCNetSDK.py – ctypes wrappers
SDK_Enum.py – enum definitions
SDK_Struct.py – ctypes structures
SDK_Callback.py – callback type definitions
```
All are imported only by `app/main.py`.

### Cross‑package interaction
* `hik‑talk` does **not** import any code from `hik‑telegram‑bot`.
* `hik‑telegram‑bot` calls the external HTTP API of `hik‑talk` (`HIK_TALK_URL`).
* Both services import standard libraries and third‑party packages (`fastapi`, `httpx`, `aiosqlite`, `python‑telegram‑bot`).

---

## 2. Dependency Summary (from docs/MODULE_DEPENDENCIES.md)

### hik‑talk (`app/main.py`)
* **Standard / third‑party imports**: `fastapi`, `uvicorn`, `asyncio`, `threading`, `queue`, `ctypes.*`, `HCNetSDK.*`, `httpx`, `logging`, `os`, `time`, `json`.
* **Internal imports**:
  * `HCNetSDK.netsdkdllpath` – path to `libhcnetsdk.so`.
  * `app.image_processor` – snapshot card processing.
* **Exports / used by**: Endpoints `/talk`, `/ws/audio`, `/api/door/open`, `/api/camera/snapshot` consumed by Front‑end JS, Telegram‑bot, tests.

### hik‑telegram‑bot (`app/main.py`)
* **Imports**: FastAPI, HTTPException, CORS, `httpx`, Pydantic models, `bot.build_bot`, `database`.
* **Dependencies**: `database` (SQLite layer), `bot` (Telegram Application).
* **Exports**: Public REST API (`/api/telegram/*`, `/api/door/open`, `/api/admin/*`). Consumed by Telegram client, Front‑end, and `hik‑talk` (door‑open proxy).

### HCNetSDK Package
* Provides ctypes definitions (`SDK_Struct`, `SDK_Enum`, `SDK_Callback`). Imported by `hik‑talk/app/main.py` and the manual test script.

### Shared Utilities
* `tools/voice_test.py` imports the same SDK but is not part of runtime.
* Placeholder modules (`audio.py`, `video.py`, `config.py`) have no imports/exports.

---

*All import relationships and external library usages are now documented in a single place for quick reference.*