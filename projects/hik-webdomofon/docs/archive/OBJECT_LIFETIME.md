# OBJECT_LIFETIME.md – Object lifetimes in the system

| Object / instance | Created by | Typical lifespan | Where it lives (module / process) |
|---|---|---|---|
| **FastAPI `app` (hik‑talk)** | Module import (`app = FastAPI(...)`) | Process lifetime (container start → stop) | Main process (`hik‑talk` service) |
| **FastAPI `app` (hik‑telegram‑bot)** | Module import (`app = FastAPI(...)`) | Process lifetime | Main process (`hik‑telegram‑bot` service) |
| **Telegram `Application` (bot)** | `build_bot()` called in FastAPI `lifespan` startup | Process lifetime (runs in background thread within the FastAPI process) | `hik‑telegram‑bot` service |
| **`VoiceSession` object** | `ws_audio` handler, per WebSocket connection | As long as the WS connection is open (≈ minutes) | In‑memory of `hik‑talk` process; accessed by both the async event loop and the background thread |
| **`threading.Thread` (`voice‑sender`)** | `VoiceSession.start()` | Until `VoiceSession.stop()` or client disconnect (joined) | Separate OS thread inside the `hik‑talk` process |
| **`queue.Queue` (`VoiceSession.q`)** | `VoiceSession.__init__` | Same as its owning `VoiceSession` (per‑client) | Shared between the async WS handler and the `voice‑sender` thread (thread‑safe) |
| **Database rows (`users`, `invites`, `call_sessions`, `admin_sessions` etc.)** | Various API handlers (`register_user`, `create_call_session`, etc.) | Persisted in SQLite file (`/app/data/users.db`) for days/weeks; individual rows may be deleted/expired (e.g., call sessions after 10 min) | Stored on disk, loaded on‑demand by `database.py` functions |
| **HTTP client sessions (`httpx.AsyncClient`)** | Short‑lived `async with` blocks in bot handlers (`_fetch_snapshot`, door open calls) | Context manager scope (a few hundred ms) | Temporary objects in the bot process |
| **WebSocket connections (client ↔ server)** | FastAPI runtime when a client initiates a WS handshake (`/talk`, `/ws/audio`) | Open until client disconnects or server error (seconds to hours) | Managed by FastAPI’s ASGI server (`uvicorn`) |
| **`CallSession` token (string)** | `db.create_call_session` when a visitor initiates a call | Valid until the owner opens the door or the session expires (≈10 min) | Stored in DB, referenced by bot handlers and the frontend via URL param |
| **`AdminSession` token** | `db.create_admin_session` when an admin opens the settings WebApp | Valid for ~1 hour (configuration) | Stored in DB, used by admin API endpoints for auth |

**Key observations**
- The only long‑living background thread is the `voice‑sender` daemon inside each `VoiceSession`. It is safely started and joined per client, preventing orphaned threads.
- All mutable shared state between the async event loop and the thread is the `queue.Queue`, which is thread‑safe, so no race conditions arise.
- Global constants (e.g., `USER_ID`, `VOICE_CHANNEL`) are immutable after import, eliminating concurrency concerns.
- Database rows provide persistence across service restarts; cleanup of expired call sessions is performed by the bot when a door‑open callback is received or when the WS disconnects.
