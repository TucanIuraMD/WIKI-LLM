# Linux Port

Migration of a Windows kiosk terminal application to Linux/Mono
(Ubuntu 24.04 LTS, Mono 6.8.0.105).

## What changed

### 1. MySQL migration (SQL Server → MySQL)

- Original app issued SQL Server queries; the DAL layer was rewritten
  (`DataBase.dll` rebuilt).
- Tables/stored procedures live in an external MySQL schema (`cashin`).
- Verified: DB integration tests 10/10 PASS.
- Key stored procedures/functions: savePayment, addMoney, get_balance,
  getSumMoney, settings (get_settings / save_setting / begin/end_store_settings),
  requisites, limitations, scheduler, main menu.

### 2. Device layer fix

- Duplicate rows in `properties4devices` and missing `CashCodeName/PrinterName`
  settings fixed; UNIQUE index `(idd_device, _key)` added.
- Device factory mapping: `Cup → CCNet` (CashCode), `printer → Citizen PPU 700`.
- Verified: device layer tests 13/13 PASS.

### 3. RSA keys fix

- The DB originally contained truncated key material → `RSA.FromXmlString()`
  threw `XmlSyntaxException` ("Ошибка ключей").
- Fix: valid 2048-bit RSA keys in .NET RSA XML format written to `cashin.keys`
  (Public ≈415 B, Secret ≈1679 B). **These are test keys — test-only.**

### 4. Mock servers + E2E

- `monitor_mock.py` (TCP), `payment_mock.py` (HTTP) for protocol-level tests.
- E2E script validates DB, devices, network, cash-in, printer, startup, cleanup:
  **13/13 PASS**.

## Verified Linux artifacts

| Assembly | Role | Status |
|----------|------|--------|
| `MainLogic.dll` | business logic | v7.0.0.0, **Linux-port rebuild, verified** |
| `CommonLib.dll` | models | rebuilt during port |
| `DataBase.dll` | MySQL DAL | rebuilt (SQL→MySQL) |
| `Devices.dll` | drivers | rebuilt |
| `LoggerLib.dll` | logging | rebuilt |
| SinkLib / Protocol / DeviceCenter / etc. | original Windows assemblies | run under Mono unmodified |

## MainLogic.dll integrity

- Version 7.0.0.0, size 319488 bytes
- SHA256 `6fd34e555367c00e20f87a0ce2c046e1215db07b02ab70c663fff96db6c724c9`
- Target .NET Framework 4.6.1, PE32/AnyCPU
- Contains Linux-port markers (`Setting.Default.PrinterName=`, `file:///C:` fix,
  `Logis is STARTED`, test hook `BanknoteStacked`)

> The old pre-port `MainLogic.dll` (v4.2.5.4) is **not** used.

## Toolchain limitation

Sources use C# 8; Mono `mcs` 6.8 supports at most C# 7.2, and the project is
SDK-style for .NET Framework with WindowsForms. A full rebuild of `MainLogic.dll`
requires Roslyn / .NET SDK with .NET Framework 4.6.1 reference assemblies
(Windows + Visual Studio, or a compatible Linux toolchain). The verified DLL must
not be replaced without a full E2E run.

Partial rebuilds that were performed on Linux: CommonLib, DataBase, Devices,
LoggerLib (from the corresponding sources).
