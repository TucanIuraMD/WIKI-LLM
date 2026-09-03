# Logic Linux — LLM Context (Main Entry)

> Public knowledge base for AI engineers maintaining the **Logic Linux** project.
> Companion docs: see `README.md`. All addresses/credentials are placeholders.

## 1. What it is

**Logic Linux** is a Linux/Mono port of a Windows self-service payment kiosk
application. The terminal:

- accepts banknotes via a **CashCode / CCNet** bill acceptor,
- prints receipts on a **Citizen PPU 700** thermal printer,
- talks to a **Monitoring Server** (TCP),
- talks to a **Payment Server** (HTTP),
- keeps its ledger in an **external MySQL** database.

Runtime environment: Ubuntu 24.04 LTS, Mono 6.8, .NET Framework 4.6.1 assemblies
(PE32/AnyCPU IL, run under Mono).

## 2. Runtime component chain (binary view)

```
Logic.exe (entry, WinForms/WebSocket UI)
 └── MainLogic.dll (business logic, scenarios, emonitor, device init)
      ├── CommonLib.dll (models: Payment, Card, ServiceTerminal, Logger)
      ├── DataBase.dll (MySQL DAL) + MySql.Data.dll
      ├── DeviceCenter.dll (device factory)
      │    └── Devices.dll / CashCodeWrapper.dll / Printer.dll (drivers)
      ├── LoggerLib.dll (file + stdout logging)
      ├── SqlSettingsProvider.dll (settings from MySQL)
      ├── SinkLib.dll (payment HTTP client)
      ├── Protocol.dll (monitoring TCP client)
      ├── SettingParser.dll (XML scenario parsing)
      ├── LogicForm.dll / AdminForm.dll (UI)
      └── Fleck.dll (WebSocket server for kiosk UI, ws://127.0.0.1:8585)
```

## 3. Key facts for maintainers

| Item | Value |
|------|-------|
| Working binary dir | `MonoDebugTemp/` in the dev checkout |
| MainLogic.dll | v7.0.0.0, 319488 B, SHA256 `6fd34e55...c9` (see `LINUX_PORT.md`) |
| Old pre-port MainLogic | v4.2.5.4 — **never use** |
| Target framework | .NET Framework 4.6.1 |
| DB | external MySQL, DB name `cashin` (see `DATABASE.md`) |
| Cash acceptor | CashCode CCNet, `/dev/ttyS0` 9600 8N1 enc 866 |
| Printer | Citizen PPU 700, `/dev/ttyS1` 19200 8N1 enc 866 |
| Monitoring | TCP `<MONITORING_SERVER>:<MONITOR_PORT>` (default 333) |
| Payment | HTTP `<PAYMENT_SERVER>:14111`, signed frames |
| UI | WebSocket on `127.0.0.1:8585` (kiosk) |

## 4. Test status (validated baseline)

```
DB             10/10 PASS
DEVICE         13/13 PASS
NETWORK        PASS
PAYMENT        PASS
MONITOR        PASS
RSA            PASS (test keys only)
CASH-IN        16/16 PASS
PRINTER        13/13 PASS
STARTUP        PASS
E2E            13/13 PASS
```

Physical CashCode / Citizen PPU 700 hardware acceptance was **not** performed in the
build environment — it remains a separate hardware step (see `TESTING.md`,
`DEPLOYMENT.md`).

## 5. Production blockers / open items

1. **RSA keys**: DB `cashin.keys` holds locally generated **test** keys (2048-bit,
   .NET RSA XML). Production requires provider-issued keys.
2. **Server addresses**: settings point at test/local hosts; production requires the
   operator's monitoring/payment endpoints.
3. **DB connection string**: hardcoded into `SqlSettingsProvider.dll` at build time;
   production must either match the MySQL credentials or rebuild the assembly.
4. **Hardware**: real CashCode/CCNet and Citizen PPU 700 were not exercised on real
   serial ports in this environment.
5. **Rebuild**: current Mono `mcs` cannot compile C# 8 sources; `MainLogic.dll`
   rebuild requires a Roslyn/.NET build environment (see `LINUX_PORT.md`).

## 6. Rules for future AI engineers

1. Never replace the verified `MainLogic.dll` (v7.0.0.0, hash in `LINUX_PORT.md`)
   without a full E2E run.
2. Never use the old pre-port `Source/MainLogic.dll` (v4.2.5.4).
3. Do not change a completed MySQL migration without a concrete documented bug.
4. Do not create new mock-device frameworks; use the built-in EMPTY stubs
   (see `DEVICES.md`).
5. Mock network/payment servers are **test-only**; they are not production acceptance.
6. Test RSA keys must never be used in production.
7. Before changing a baseline, create a backup.
8. After substantial changes run the E2E script (see `TESTING.md`); all checks must PASS.
9. Hardware acceptance is separate from mock tests.
10. Distinguish change types: source change, binary change, DB change, configuration
    change — each has its own validation procedure.

## 7. Document map

| Doc | Content |
|-----|---------|
| `ARCHITECTURE.md` | components, flows, processes |
| `LINUX_PORT.md` | migration history, artifacts, hashes |
| `DATABASE.md` | external MySQL contract, tables, procedures, schema fixes |
| `DEVICES.md` | DeviceCenter, CashCode, Citizen PPU 700, COM layer, EMPTY stubs |
| `NETWORK.md` | network overview |
| `PAYMENT.md` | payment protocol, signed frames, state codes |
| `MONITORING.md` | monitoring protocol |
| `INFO_XML.md` | info.xml structure and parsing |
| `TESTING.md` | tests, mocks, E2E |
| `TROUBLESHOOTING.md` | diagnostics |
| `DEPLOYMENT.md` | deployment outline, udev, systemd |
