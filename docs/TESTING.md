# Testing and E2E

This page describes the test tooling used during the Linux port. All of it is
**test-only**; mock servers and EMPTY device stubs are not production components.

## Test executables

| Executable | Checks | Purpose |
|------------|--------|---------|
| `DbIntegrationTest.exe` | 10 | DAL against a local MySQL: settings, balance, savePayment, get-not-sent payments, addMoney, ticket number, tasks, services, main menu |
| `DeviceLayerTest.exe` | 13 | device data from DB: GetDevices, GetDeviceByName (CCNet / Citizen), device properties, no duplicates |
| `CashInE2ETest.exe` | 16 | cash-in flow via DAL: AddMoney → money_history, SavePayment → ticket, multi-denominations, invalid denom, ledger consistency, balance roundtrip, status, cleanup |
| `PrinterE2ETest.exe` | 13 | PrinterEMPTY stub: Open, GetState (LentaStatus OK), Print (empty document), Info, Close, reopen |

All executables are test-only and are **not** part of a production install.

## Mock servers (test-only)

- `monitor_mock.py` — TCP server emulating the monitoring protocol (ACKs).
- `payment_mock.py` — HTTP server emulating the payment protocol: decodes the
  signed request, replies with plain XML result frames (2000/7000/10000, Info
  response for Info frames).

Mock servers validate wire-format compatibility only. They are **not** a
substitute for real server acceptance.

## E2E script

The E2E script runs phases: environment (MySQL/Mono/Xvfb) → DB tests → device
tests → start mocks → network checks → cash-in tests → printer tests → start
`Logic.exe` and wait for `Logis is STARTED` → monitoring connection check →
report.

```
DB             : 10/10 PASS
DEVICE         : 13/13 PASS
NETWORK        : PASS
PAYMENT        : PASS
MONITOR        : PASS
CASH-IN        : 16/16 PASS
PRINTER        : 13/13 PASS
STARTUP        : PASS
E2E            : PASS
```

Run with: `bash <repo>/start_logic_linux_test.sh` (dev checkout layout).

## Realistic caveats

- Physical CashCode / Citizen PPU 700 hardware acceptance was **not** performed in
  the build environment — no real devices were attached.
- RSA tests used locally generated **test** keys, not production keys.
- Real monitoring/payment servers were not contacted.
- Info flow was verified end-to-end against the payment mock: GetInfo task
  reached **Success**; settings, operators, services and limitations were
  persisted to the DB (see `INFO_XML.md` and `LINUX_PORT.md`).

## How to reproduce

1. Start MySQL; provision schema/data in a local test database.
2. Point test settings at local mocks (`127.0.0.1`).
3. Run the E2E script from the dev checkout.
4. Inspect `/tmp/logic_test_logs/<timestamp>/` for reports.
