# Troubleshooting

## Startup / runtime

| Symptom | Cause | Solution |
|---------|-------|----------|
| "Ошибка ключей: System error." / `XmlSyntaxException` | invalid RSA key material in `cashin.keys` | Write valid .NET RSA XML (UTF-8) into `Public`/`Secret`; see `DATABASE.md` |
| Monitoring "Connection refused" | monitoring server unreachable | start local mock (test) or configure real `MonitorURL`; check firewall |
| Payment channel unavailable | payment server unreachable | start local mock or configure real `PaymentUri` |
| CashCode Open() COM-port exception | `/dev/ttyS0` missing / no permission | verify physical port, udev rules, `dialout` group; on dev use DB-layer tests |
| Printer Open() error | `/dev/ttyS1` missing | same; for tests use the PrinterEMPTY stub |
| "Logis is STARTED" never appears | DB unreachable / config | check MySQL, connection string, settings |
| Log file not where expected | LoggerLib uses `<appDir>\logs` (backslash in name) | look under the dev checkout's log dir with the literal backslash name |
| "Fatal error, Printer Error" block | printer unavailable at startup | expected on dev without hardware; verify printer in production |

## Database

| Symptom | Cause | Solution |
|---------|-------|----------|
| savePayment returns NULL/0 | wrong params/types | verify call: GUID id, decimal amounts, service varchar |
| cash total ≠ payments total | money/payments out of sync | use `checkSumMoneyAndPayments`; inspect `money`/`money_history` |
| No MySQL connection | bind/port/grant | check bind `127.0.0.1`, grants for `<DB_USER>` on the schema |
| Balance unchanged after AddMoney | balance is a separate table | `AddMoney` writes `money`/`money_history`; `GetBalance` reads `balance` after `SetBalance` |

## Tests

| Symptom | Cause | Solution |
|---------|-------|----------|
| Cash-in E2E amounts accumulate between runs | test not idempotent in absolute amounts | tests use deltas; reset `money_history`/`money` if needed |
| Printer E2E shows MessageBox on print | PrinterEMPTY displays contents | use an empty document or Xvfb |
| `mono` not found | Mono not installed | `apt install mono-runtime mono-complete` |

## Network

| Symptom | Cause | Solution |
|---------|-------|----------|
| Mock does not accept connections | port busy | `ss -tlnp | grep -E ':333|:14111'`, kill process |
| Repeated "Сообщение ... повторное" | resend without ACK | ensure the peer ACKs (type+number) |
| Payment mock "XML parse error" | unexpected request format | verify frame XML root tags (`<process>`, `<check>`, `<info>` ...) |

## Files

| Symptom | Solution |
|---------|----------|
| Stray `C:\dabaseDLL_*.txt` files | debug artifact of DAL; safe to ignore/remove |
| Log file named `...\logs\...` (backslash) | normal LoggerLib behaviour on Linux |

## Tools

```bash
# ports
ss -tlnp | grep -E ':333|:14111|:3306|:8585'
# app log (dev checkout)
tail -f '<checkout>/MonoDebugTemp\logs/<date>.log'
# verified MainLogic hash
sha256sum <checkout>/MonoDebugTemp/MainLogic.dll   # → 6fd34e55... (see LINUX_PORT.md)
# DB settings
mysql -u <DB_USER> -p cashin -e "SELECT _name,_value FROM settings"
```
