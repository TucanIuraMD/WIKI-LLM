# Frontend Overview (Mini App)

This document consolidates the user‑facing parts of the system: the Telegram Mini App UI, the video subsystem and the audio subsystem.

---

## 1. Telegram Mini App (from docs/08_MINI_APP.md)

### Purpose
Telegram Mini App is the main user interface of the system. Through the Mini App a user gets access to all door‑phone functions. The bot is used only to launch the app, send notifications and quick commands.

### Core Principles
* No business logic resides in the Mini App – all operations are performed via the Backend API.
* The app is responsible for displaying data, handling user interaction and forwarding requests to the Backend.

### UI Structure
#### Main screen
* Shows video stream, connection status, camera status and primary actions.
* Available actions: open door, start conversation, end conversation, open menu.

#### Conversation screen
* Push‑To‑Talk, connection state indicator, recording indicator, end call.

#### History screen
* Shows visits, door opens, calls, system events with search, filter and sort.

#### Invitations screen
* Create invitation, set expiry, limit usage, delete invitation.

#### Profile screen
* Shows user name, role, Telegram ID, user settings.

#### Settings (admin only)
* Change system parameters, manage users, view audit log, perform admin actions.

#### Interaction with Backend
* Uses REST API for data fetching and state changes.
* Uses WebSocket for events, state updates and voice communication.

#### Authorization
* Performed via Telegram Login, Backend validates signature, user and permissions.

---

## 2. Video Subsystem (from docs/09_VIDEO.md)

### Purpose
Provides real‑time video view from the Hikvision camera via the web interface.

### Architecture
```
Hikvision Camera → RTSP → go2rtc → WebRTC / HLS → Mini App
```
Only the Sub Stream (Channel 102) is used. Channel 101 is reserved and not used.

### go2rtc responsibilities
* Fetch RTSP stream.
* Convert to WebRTC and HLS.
* Publish endpoints for the Mini App.

### Backend role
* Does **not** handle video.
* Supplies stream configuration and URLs to the Mini App.

### Mini App workflow
* Requests stream configuration via REST.
* Connects to the video stream (WebRTC preferred, HLS fallback).
* Reconnects on failures.

### Performance & Limitations
* Single video source, Channel 102 only.
* Video is independent of audio subsystem.

---

## 3. Audio Subsystem (from docs/10_AUDIO.md)

### Purpose
Provides bidirectional voice communication between the user and the Hikvision camera using Push‑To‑Talk.

### Architecture diagram
```
Telegram Mini App → WebSocket (/ws/audio) → Audio Session → Audio Queue (20 ms) → VoiceSession → HCNetSDK → Hikvision Camera
```

### Components
* **Mini App** – captures microphone, sends audio, plays incoming audio, shows conversation state.
* **WebSocket** – endpoint `/ws/audio` for two‑way audio.
* **Audio Queue** – buffers incoming audio frames (20 ms each).
* **VoiceSession** – per‑connection session managing the queue and SDK channel.
* **HCNetSDK** – used **only** for voice talk.

### Codec
* G.711 μ‑law (only codec used).

### Push‑To‑Talk flow
1. User presses talk button.
2. Mini App opens WebSocket.
3. Backend creates `VoiceSession`.
4. HCNetSDK starts voice talk.
5. Audio frames are sent (µ‑law, 160 bytes per 20 ms).
6. User releases button → recording stops.
7. Session closes, resources freed.

### Constraints
* Only one active `VoiceSession` at a time.
* One authorised HCNetSDK session.
* Frame size fixed at 160 bytes (20 ms).

---

## 4. Cross‑Component Links
* The Mini App interacts with both the **Video** and **Audio** subsystems via the same Backend API and WebSocket layer.
* All UI actions ultimately result in HTTP calls defined in **docs/06_API.md**.
* Detailed runtime behaviour is described in **docs/RUNTIME_FLOW.md**.

---

*This consolidated overview replaces the three separate documents (08_MINI_APP.md, 09_VIDEO.md, 10_AUDIO.md) while keeping their full information.*