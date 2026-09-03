# Deployment

Production deployment outline. The database is external and provisioned
separately (`DATABASE.md`); the application connects to it.

## Target environment

- Ubuntu 24.04 LTS (x86_64)
- Mono 6.8 runtime packages
- MySQL 8.x bound to `127.0.0.1`
- dedicated service user (e.g. `logic`) in group `dialout`
- app directory (e.g. `/opt/logic/run`)

## Runtime file set (Linux build)

`Logic.exe` + the Linux-port assemblies (see `LINUX_PORT.md` for the verified
set and hashes) + required third-party libraries (MySQL connector, JSON, crypto,
WebSocket, zip, BCL backports). Exclude test executables, mock servers, symbols
(PDB/MDB), `_originals`, Windows leftovers.

## Database provisioning

1. Provision MySQL schema and reference data for the target schema (external,
   outside this repository).
2. Apply the documented schema items if needed (see `DATABASE.md`:
   `service` columns, `limitation`, `save_setting`, `addLimitation` alignment).
3. Create/limit DB user grants to the application schema only.
4. Configure settings in MySQL: `MonitorURL`, `MonitorPort`, `PaymentUri`,
   terminal `PointId`/`ClientId`, `CashCodeName`/`PrinterName`, `LogRequest=false`.
5. Install production RSA keys into `cashin.keys` (Public/Secret) — see
   `DATABASE.md`. Never reuse test keys.

## Serial port / udev

Devices: CashCode CCNet on `/dev/ttyS0`, Citizen PPU 700 on `/dev/ttyS1`
(configurable via `properties4devices`; see `DEVICES.md`).

udev rules (example):

```
KERNEL=="ttyS*", MODE="0660", GROUP="dialout"
KERNEL=="ttyUSB*", MODE="0660", GROUP="dialout"
KERNEL=="ttyACM*", MODE="0660", GROUP="dialout"
```

Service user must be a member of `dialout`.

## Running the application

Run the entry assembly under Mono with the kiosk display available:

```
/usr/bin/mono /opt/logic/run/Logic.exe
```

The app reaches the "Logis is STARTED" state and starts monitoring/payment
channels, periodic tasks (GetInfo), and the WebSocket UI.

## systemd (outline)

A unit file runs the app as the service user after MySQL and network are up,
restarts on failure, and restricts device access to the serial ports needed.
The UI process expects a display; use an appropriate X/Wayland setup or kiosk
display server.

## Ports

- out: monitoring TCP (333 by convention)
- out: payment HTTP (14111 by convention), optionally via proxy
- in (optional): WebSocket UI `127.0.0.1:8585`
- in: MySQL `127.0.0.1:3306`

## Post-deploy checks

1. `sha256sum` of `MainLogic.dll` matches the verified hash (`LINUX_PORT.md`).
2. Preflight script passes (see `TESTING.md`).
3. Logic startup: monitoring ACKs, payment channel configured, GetInfo task runs.
4. Hardware acceptance on real CashCode/Citizen devices is a separate step.
5. E2E regression (dev environment) remains 13/13.
