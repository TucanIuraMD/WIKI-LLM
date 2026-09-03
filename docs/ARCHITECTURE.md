# Architecture

## Component map

```
┌───────────────────────────────────────────────────────────┐
│                      Logic.exe (entry)                     │
│  WinForms/WebSocket UI ←→ LogicForm.dll (FormManagerSocket)│
└──────────────────────────┬────────────────────────────────┘
                           │
┌──────────────────────────▼────────────────────────────────┐
│                   MainLogic.dll                           │
│  • scenarios (XML via SettingParser)                       │
│  • payment flow: cash-in → SavePayment → send              │
│  • emonitor: monitoring channel                            │
│  • sender thread: unsent payments                          │
│  • device init: BillAcceptor, PrinterDevice                │
└──┬─────────┬──────────┬──────────┬────────────────────────┘
   ▼         ▼          ▼          ▼
CommonLib  DataBase.dll DeviceCenter SinkLib.dll
(models)   (MySQL DAL,  (device      (HTTP payment
           MySql.Data)  factory)     client)
                │          │          │
                │          ▼          ▼
                │     Devices.dll    Payment server
                │     CashCodeWrapper (HTTP POST)
                │     Printer.dll
                ▼
           MySQL `cashin`         Protocol.dll
           (settings, keys,       (monitoring TCP)
            devices, payments,
            money, messages)

LoggerLib.dll (log dir + stdout)
SqlSettingsProvider.dll (settings from MySQL settings table)
```

## Main classes / responsibilities (MainLogic)

| Class | Responsibility |
|-------|----------------|
| `IBP.MainLogic` | main controller: Init(), scenarios, timers, device init |
| `IBP.Emonitor` | monitoring channel wrapper (TCP) |
| sender class | payment sending thread (SinkLib Formatter) |
| `IBP.TaskScheduller` | periodic tasks: GetInfo, SendMessage (check settings), CheckSum |
| `IBP.FormManagerSocket` | WebSocket UI bridge |
| scenario step classes | UI steps: input amount, accept banknotes, print, etc. |

## Data flow — cash-in (simplified)

1. Scenario step puts the terminal in accept mode; `BillAcceptor` accepts a banknote.
2. Currency event → `DAL.AddMoney(nominal)` → `money` + `money_history`.
3. After scenario completion → `SavePayment(ref payment)` →
   DB `savePayment` → `payments` row + ticket number.
4. Sender thread picks unsent payments → `Formatter.Process(frame)` → HTTP POST to
   the payment server URI.
5. Response parsed → payment state (e.g. 2000 Accepted) → `sent=1`.

## Data flow — monitoring

1. `Emonitor` → TCP connect to monitoring host.
2. Send Initialization, CurrentDateTime, and unsent messages from the DB.
3. Server responds with ACKs and may push commands (config, restart, etc.).
4. Commands are dispatched to callbacks.

## Data flow — Info / configuration

1. Periodic GetInfo task requests an Info frame from the payment server.
2. Response is verified and saved to `info.xml` (or `infoshort.xml`).
3. `parseInfoXml` applies header, operators, services, limitations and other
   sections into settings and DB (see `INFO_XML.md`).

## Notable port characteristics

- **UI**: `Design=WebSocket` → `FormManagerSocket` + WebSocket server on
  `127.0.0.1:8585`.
- **Logging**: LoggerLib writes into `<appDir>\logs\yyyy-MM-dd.log` (directory name
  literally contains a backslash on Linux) and stdout.
- **Settings**: read from MySQL `cashin.settings` through `SqlSettingsProvider`
  (`IBP.Setting`); defaults are declared in `Setting.cs`.
- Several components are original Windows assemblies that run unmodified under Mono
  (e.g. SinkLib, Protocol, DeviceCenter); only a subset was rebuilt for Linux
  (see `LINUX_PORT.md`).
