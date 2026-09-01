# Проект: Web Push-To-Talk → Hikvision Camera

## Цель

Передача голоса из браузера (ПК, Android, iPhone) на камеру Hikvision через двустороннюю аудиосвязь (Voice Talk).

Схема:

Browser
↓
WebSocket
↓
FastAPI
↓
HCNetSDK
↓
Hikvision Camera

---

# Оборудование

Камера:

Model:
DS-2CD2432F-I

Firmware:
V5.4.5 build 170123

IP:
192.168.80.25

SDK Port:
8000

User:
admin

---

# Сервер

Ubuntu/Debian Linux.

Python venv:
~/hik-venv

HCNetSDK-python:
pip install HCNetSDK-python

Используется родная библиотека:
/root/hik-venv/lib/python3.13/site-packages/HCNetSDK/Libs/linux/libhcnetsdk.so

---

# HCNetSDK

Работает:

NET_DVR_Login_V40
NET_DVR_StartVoiceCom_MR_V30
NET_DVR_VoiceComSendData
NET_DVR_StopVoiceCom

Ошибка:

29 =
NET_DVR_DVRVOICEOPENED

Voice talk channel on the device has been occupied.

---

# Очень важные выводы проекта

Камера НЕ принимает:

- raw PCM
- G711A

Камера принимает:

G711U (μ-law)

После перехода на G711U голос стал идеальным.

---

# Формат аудио

Browser:
48 kHz PCM
1 channel

Server:
8 kHz
G711U
160 byte frames
20 ms interval

---

# Рабочая схема

48 kHz PCM
↓
 downsample to 8 kHz PCM
↓
PCM → G711U
↓
160 bytes
↓
every 20 ms
↓
NET_DVR_VoiceComSendData()

---

# Размеры кадров

320 bytes PCM
↓
160 samples
↓
160 bytes G711U

Отправлять:
160 bytes каждые 20 ms.

---

# Серверный цикл

```python
while len(buffer) >= 320:
    raw_chunk = bytes(buffer[:320])
    del buffer[:320]

    ulaw = pcm_to_ulaw(raw_chunk)

    sdk.NET_DVR_VoiceComSendData(
        handle,
        ulaw,
        160
    )
```
