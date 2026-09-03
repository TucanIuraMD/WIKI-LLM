# External MySQL Database

> The database is a **separate component**. It is not part of the application
> repository. The application connects to an already-provisioned MySQL database.
> Credentials shown here are placeholders.

## Contract

| Parameter | Value (example) |
|-----------|-----------------|
| DBMS | MySQL 8.x, utf8mb4 |
| Host | `127.0.0.1` (recommended bind) |
| Port | 3306 |
| User | `<DB_USER>` |
| Password | `<DB_PASSWORD>` (not committed anywhere) |
| Database | `cashin` |
| Connection options | `CharSet=utf8mb4` |

The connection string is compiled into `SqlSettingsProvider.dll` at build time
(`ConnectionStringHelper`). To change it either align the MySQL account with the
compiled value or rebuild the assembly.

## How the application uses the DB

- **Settings**: read via `get_settings` → table `settings`; `SqlSettingsProvider`
  exposes them as `IBP.Setting` properties with code defaults as fallback.
- **Keys**: `keys` table holds RSA material: `Public`, `Secret`, and channel
  password `Psw` (the latter used only with a specific authentication mode).
- **Devices**: `devices` + `properties4devices` describe peripherals and COM
  parameters (see `DEVICES.md`).
- **Ledger**: `payments`, `money`, `money_history`, `balance`.
- **Messaging**: `messages`, `message_big`, `messagetomonitor`.
- **Reference data**: services, operators, requisites, limitations, states,
  languages, tasks, scenarios, cheques, barcodes, etc.

## Main tables

| Table | Purpose |
|-------|---------|
| `settings` | application settings (`_name`, `_value`), section model |
| `keys` | RSA keys: `id_key ∈ {Public, Secret, Psw}`, `_key` BLOB |
| `devices` | devices (`id_device`, `type`, `description`, `name`) |
| `properties4devices` | device properties, UNIQUE `(idd_device, _key)` |
| `payments` | payments ledger |
| `money` / `money_history` | cash totals / banknote insert history |
| `balance` | balance row |
| `service` / `operators` / `requisites` / `limitation` | services & relations |
| `messages` family | monitoring messages |
| `tasks_scheduler` / `attr4tasks` | periodic tasks |
| `SettingsSection` | setting sections |

## Stored procedures / functions (summary)

| Name | Purpose |
|------|---------|
| `savePayment(...)` | save a payment, returns ticket row id |
| `getSumMoney` | sum of cash |
| `addMoney` / DAL.AddMoney | record a banknote insert |
| `get_balance` | current balance |
| `get_settings` | settings for a section |
| `begin_store_settings` / `save_setting` / `end_store_settings` | persist a settings section |
| `addLimitation`, `addServiceRequisite`, `addOperatorRequisite`, `addRequisites` | service/operator data |
| `checkSumMoneyAndPayments` | cash/payments reconciliation |
| `saveIncasso`, `saveModule`, `MMPS_updateMainMenu` | admin/aux ops |

## Schema fixes applied during Linux port (Info flow)

### `service` — added 4 columns (aligned with the original schema)

| Column | Type | Null / Default | Purpose |
|--------|------|----------------|---------|
| `idd_operator` | INT | NULL | owning operator id |
| `check_timeout` | INT | NOT NULL DEFAULT 30 | check timeout (seconds) |
| `external_id` | VARCHAR(100) | NULL | external service id |
| `idd_requisite` | INT | NULL | link to `requisites.id_requisite` |

Used by `DAL.SaveServicesFromInfo`, `addServiceRequisite`, `GetUsluga`.

### `limitation` — table created (absent during migration)

Columns: `id_limitation` PK auto-increment, `idd_service` (index → `service.id_service`),
`start_date`, `min_rate`/`max_rate`/`min_amount`/`max_amount` DECIMAL(18,2),
`type_com`, `type_limit`, `flag_edit` default 0.

Used by `SaveServicesLimits` → `addLimitation` for Info flows with
protocol version ≥ 2.3.2.

### `save_setting` — IF EXISTS/UPDATE/INSERT with `active=0`

Background: MySql.Data 8.0.26 on Mono triggered `KeyNotFoundException('0')` for
`INSERT ... ON DUPLICATE KEY UPDATE` results (charset index 0). The rewrite to
`IF EXISTS → UPDATE ... , active=0; ELSE INSERT`:

- avoids the ODKU result-set path (no KeyNotFoundException), and
- resets `active=0`, preserving the original `begin_store_settings` /
  `end_store_settings` semantics (otherwise the end step would delete rows that
  were just re-saved).

### `addLimitation` — parameter order aligned with DAL call

MySql.Data on Mono passes stored-procedure parameters **positionally**. The SP
parameter order was aligned with the order used by `DAL.SaveServicesLimits`
(`_minRate, _maxRate, _minAmount, _maxAmount, _typeCom, _startDate, _alias,
_typeLimit`).

## Notes

- `keys._key` is a BLOB of UTF-8 bytes of .NET RSA XML.
- `payments._status` is an int status (e.g. -1 undefined, 0 new, 2 ready,
  2000 accepted).
- `payments.sent` is 0/1.
- A test copy `cashin_test` may exist for isolated testing; production grants
  should not include it.
