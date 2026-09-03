# Monitoring Protocol

The terminal maintains a persistent TCP connection to a monitoring server.

## Transport

- **Endpoint**: `<MONITORING_SERVER>:<MONITOR_PORT>` (default 333), configured via
  `MonitorURL` / `MonitorPort` settings.
- Raw TCP, client-initiated, persistent connection.

## Framing

- 4-byte little-endian length prefix + package bytes.
- Package: `flags(1)` + `code(1)` + `number(4 LE)` + optional body
  (when `NewVersion` flag 0x80 is set).

Flags:

| Bits | Meaning |
|------|---------|
| 0-1 | PackageType: 0 Initialization, 1 Message, 2 Command, 3 Pool |
| 3 | Ack (0x08) |
| 4 | Compressor (0x10) — deflate when body > 350 bytes |
| 7 | NewVersion (0x80) |

## Message flow

1. Client connects.
2. Client sends Initialization, then CurrentDateTime, then unsent messages from
   the DB.
3. Server responds with ACKs (type + number); may push Command packages.

## Message codes (partial)

| Code | Name |
|------|------|
| 1 | VersionLogic |
| 2 | VersionFlash |
| 3 | VersionService |
| 4 | ChequeLenta |
| 5 | CashCode |
| 6 | CurrentDateTime |
| 10 | Currency |
| 20 | Balance |
| 99 | TextMessage |
| 100 | MetkaInfo |
| 300 / 301 | Response / ResponseEx |

TextMessage body: `[pointState:u8][messageType:u8][UTF8 text]`.
CurrentDateTime body: Int64 LE `DateTime.Ticks`.

## Commands (server → client)

CheckUpdate, DownLoadSettingsXml, UploadZip, RestartLogic, ExecuteSql,
SetBlock, SetConfig, etc.

## Client implementation

- `IBP.Emonitor` (MainLogic) wraps `ClientFormatter` (Protocol.dll).
- `ClientFormatter.Start()` runs pool + command threads.

## Test mock

A test mock server (`monitor_mock.py`) accepts connections, parses packages,
replies with ACKs and logs traffic (see `TESTING.md`).