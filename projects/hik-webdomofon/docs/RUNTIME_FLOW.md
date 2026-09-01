# RUNTIME_FLOW.md – Key execution sequences

1. **Application start (hik‑talk)**
   - Docker `CMD` runs `uvicorn hik_talk.app.main:app --host 0.0.0.0 --port 9000`.
   - FastAPI creates the `app` object, registers routes, loads environment variables via `app/config.py`.
   - No DB connection needed – the service is stateless apart from the SDK library.

2. **WebSocket `/talk` (legacy two‑way audio)**
   - Client (browser) opens WS to `/talk`.
   - `talk(ws)` handler accepts the socket, calls `sdk.NET_DVR_StartVoiceCom_MR_V30` to open a voice channel.
   - Incoming binary messages contain PCM audio (48 kHz, 16‑bit). The handler buffers them, converts to G711U (`pcm_to_ulaw` → `linear2ulaw`), and sends each 160‑byte frame to the SDK via `NET_DVR_VoiceComSendData`.
   - The loop maintains a strict 20 ms cadence (`next_send` timing) to match the camera’s expectations.
   - On disconnect or error the SDK channel is closed with `NET_DVR_StopVoiceCom`.

3. **WebSocket `/ws/audio` (modern push‑to‑talk)**
   - Client opens WS to `/ws/audio`.
   - `ws_audio` creates a `VoiceSession` instance (holds a thread‑safe `queue.Queue`).
   - Starts an asyncio heartbeat task that periodically sends a ping JSON message.
   - Client sends JSON control messages (`talk_start`, `talk_stop`) and binary PCM frames.
   - On `talk_start` the session spawns a daemon `threading.Thread` running `VoiceSession._worker`:
     * Opens SDK voice channel (`_open_channel`).
     * Pulls µ‑law frames from the queue (`_pump_frames`) and forwards them to the SDK (`NET_DVR_VoiceComSendData`).
     * Implements retry‑back‑off for SDK error 29 (channel already occupied).
     * Closes the channel on finish (`_close_channel`).
   - PCM frames from the client are converted to µ‑law (`pcm_to_ulaw`) and placed into the queue.

4. **Snapshot endpoint**
   - GET `/api/camera/snapshot?processed=0|1`.
   - `_fetch_raw_snapshot` first tries the camera’s ISAPI JPEG snapshot URL (basic auth). If that fails, it falls back to `go2rtc` static frame endpoint (`/api/frame.jpeg?src=camera_sub`).
   - If `processed=1` the raw JPEG is passed to `app.image_processor.process_snapshot` which:
     * Rotates according to EXIF, rescales to 1280 px width, overlays a semi‑transparent black panel with timestamp, and recompresses to ≤300 KB.
   - The result is returned as `image/jpeg`.

5. **Telegram Bot flow (visitor → owner)**
   - Visitor sends a message to the bot. Bot enters state `awaiting_apartment`.
   - Visitor provides apartment number → bot validates, looks up owners via `db.get_notifiable_users`.
   - Bot fetches a processed snapshot from hik‑talk (`/api/camera/snapshot?processed=1`).
   - Bot creates a call‑session record in SQLite (`db.create_call_session`).
   - For each owner:
     * Builds an InlineKeyboard with a WebApp button (`doorphone.html?session=<token>`).
     * Sends a photo (snapshot) or a plain message with the keyboard via Telegram API.
   - Owner clicks the WebApp button → opens `doorphone.html` which connects to `/ws/audio` and starts the push‑to‑talk flow.
   - Owner can also press the “Open Door” button (callback data `d:<visitor_id>:<house_code>`). The bot receives the callback, validates permissions, forwards a POST to hik‑talk `/api/door/open`, logs the action, and edits the Telegram message to reflect success/failure.

6. **Door open endpoint (hik‑talk)**
   - POST `/api/door/open` (currently a placeholder) logs the request and returns `{"status":"ok"}`. Real door control is performed elsewhere (future work).

7. **Shutdown**
   - Docker container stop sends SIGTERM, `uvicorn` stops accepting new connections, existing WS connections are closed, threads terminate, and the process exits.

---

**Key points**
- All audio processing (PCM → µ‑law) happens in Python; the SDK only receives µ‑law frames.
- The queue size (8) limits buffering to ~160 ms of audio, preventing unbounded memory growth.
- The heartbeat task keeps the WS alive for clients behind proxies.
- Snapshot processing is CPU‑light (Pillow) and cached per request.
- Telegram‑bot ↔ hik‑talk communication is pure HTTP (via `httpx`).
