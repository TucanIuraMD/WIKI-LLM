# SDK_FLOW.md – HCNetSDK interaction map

| Call site (function) | SDK function | Arguments (high‑level) | Purpose | Thread / async context |
|---|---|---|---|---|
| `talk` (WS `/talk`) | `NET_DVR_StartVoiceCom_MR_V30` | `user_id=USER_ID`, `voice_channel=VOICES_CHANNEL`, `callback=None` | Open a two‑way voice channel on the camera for the legacy WS endpoint | FastAPI event loop (single thread) |
| `talk` (inside frame loop) | `NET_DVR_VoiceComSendData` | `handle`, `ulaw_frame` (160 bytes), `frame_len=160` | Send a single µ‑law audio frame to the camera | FastAPI event loop |
| `talk` (cleanup) | `NET_DVR_StopVoiceCom` | `handle` | Close the voice channel when the client disconnects | FastAPI event loop |
| `VoiceSession._open_channel` | `NET_DVR_StartVoiceCom_MR_V30` | Same as above, but called from the background thread handling `/ws/audio` | Open a voice channel for the modern push‑to‑talk flow | Background `voice‑sender` daemon thread |
| `VoiceSession._pump_frames` | `NET_DVR_VoiceComSendData` | `self.handle`, µ‑law frame from queue | Push encoded audio frames to the camera, respecting 20 ms cadence | Background thread |
| `VoiceSession._close_channel` | `NET_DVR_StopVoiceCom` | `self.handle` | Close the voice channel when the session ends or on error | Background thread |
| `VoiceSession._notify_error` (on SDK error 29) | `NET_DVR_GetLastError` (via wrapper) | – | Retrieve the last SDK error code to decide on retry/back‑off | Background thread |
| (Other SDK symbols) | Various constants (`NET_DVR_DVRVOICEOPENED`, etc.) | – | Used for error‑code comparison in the retry logic | Background thread |

**Flow summary**
1. When a client issues `talk_start` (or connects to `/talk`), the server calls `NET_DVR_StartVoiceCom_MR_V30` to allocate a voice channel.
2. PCM audio from the client is converted to µ‑law (G711U) and fed to `NET_DVR_VoiceComSendData` at 20 ms intervals.
3. If the SDK returns error 29 (`NET_DVR_DVRVOICEOPENED` – channel already in use), the code backs off using the `WS_RETRY_DELAYS` tuple and retries up to 5 times.
4. On normal termination (`talk_stop` or WS disconnect) the channel is closed with `NET_DVR_StopVoiceCom`.
5. All SDK calls are made from the thread that owns the `VoiceSession` (either the FastAPI event loop for `/talk` or the dedicated daemon thread for `/ws/audio`).