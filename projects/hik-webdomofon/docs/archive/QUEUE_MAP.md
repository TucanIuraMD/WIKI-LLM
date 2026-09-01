# Queue Map

| Queue variable | Producer(s) | Consumer(s) | Maxsize | Data type (elements) |
|---|---|---|---|---|
| `VoiceSession.q` (`self.q`) | `VoiceSession.push_pcm` (called when client sends PCM bytes) | `VoiceSession._pump_frames` (running in `voice-sender` thread) | `WS_QUEUE_MAXSIZE` = 8 | `bytes` (already µ‑law‑encoded G711U frames, each 160 bytes) |