# Devices

## DeviceCenter factory

`IBP.BaseFactory` creates devices by type + description:

- `GetCashCode("CCNet")` → CCNet bill acceptor driver
- `GetPrinter("Citizen PPU 700")` → Citizen thermal printer driver
- Each factory locates classes annotated with `[DescriptionMpl("...")]` inside the
  relevant assembly.

Device selection at runtime is driven by MySQL:

| Logical name (setting) | devices.type | devices.description | Driver class |
|------------------------|--------------|---------------------|--------------|
| `Cup` (CashCodeName) | CashCode | `CCNet` | CCNet driver (CashCodeWrapper.dll) |
| `printer` (PrinterName) | Printer | `Citizen PPU 700` | Citizen driver (Devices.dll) |

## Serial (COM) layer

`COMPortWrapper` wraps `System.IO.Ports.SerialPort` (Mono). Defaults:
COM1, 19200, 8N1, encoding 866; overridden by `properties4devices`
(e.g. `ComPort=/dev/ttyS0`).

Port discovery uses `SerialPort.GetPortNames()` (Linux: `/dev/ttyS*`,
`/dev/ttyUSB*`).

## CashCode / CCNet

| Parameter | Value |
|-----------|-------|
| Interface | RS-232 (`/dev/ttyS0`) or USB→serial |
| BaudRate / DataBits / Parity / StopBits | 9600 / 8 / None / One |
| Encoding | 866 |
| Permissions | user in `dialout`, udev `MODE="0660" GROUP="dialout"` |

Driver highlights:
- `CashPoll()` polls the device; `CashPollStop()`, `CashReset()`,
  `GetBillTable()`.
- `Open()` opens the COM port via `COMPortWrapper`.
- Events surface to MainLogic: `OnError`, `OnCurrency` (banknote inserted →
  `AddMoney`), `OnEscrow`, `OnAccepting`, `OnLog`.
- `Autodetect()` iterates available COM ports and logs
  `billacc_avtodetect.log`.

## Citizen PPU 700

| Parameter | Value |
|-----------|-------|
| Interface | RS-232 (`/dev/ttyS1`) or USB→serial |
| BaudRate / DataBits / Parity / StopBits | 19200 / 8 / None / One |
| Encoding / PaperWidth / CodeTable | 866 / 80 / 7 |
| Permissions | same dialout model |

Driver highlights:
- `GetState()` polls registers with `0x10 0x04 0x01..0x06` and decodes flags
  (paper status, cover, errors).
- `Print(PrinterDocument)` — ESC/POS-like printing (text/barcode/image objects).
- `PrinterInfo` describes capabilities (fiscal?, image, barcodes).

## Built-in EMPTY stubs (test-only, do not ship to production)

| Class | DescriptionMpl | Behaviour |
|-------|----------------|-----------|
| `CashCodeEmpty` | `"EMPTY"` | no-op polling, empty bill table; stub accept hook |
| `PrinterEMPTY` | `"Заглушка"` | Open logs; GetState → LentaStatus OK; Print collects lines; no hardware |

Production DB rows point to real drivers (`CCNet`, `Citizen PPU 700`), not to the
stubs. Stubs are reachable via the factory when description matches.

## Notes

- Without physical hardware, driver `Open()` raises a COM-port exception; this is
  expected and logged. The application continues running.
- Hardware acceptance is a separate step on real devices (see `TESTING.md`,
  `DEPLOYMENT.md`).
