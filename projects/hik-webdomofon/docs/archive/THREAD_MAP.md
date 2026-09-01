# Thread Map

| Thread name | Started by | Target function | Purpose |
|---|---|---|---|
| `voice-sender` (daemon) | `VoiceSession.start()` | `VoiceSession._worker` | Opens HCNetSDK voice channel, pulls encoded µ‑law frames from internal queue, sends them to camera, handles SDK error 29 retries, closes channel, notifies client. One thread per active `/ws/audio` session. |