# Info XML

`info.xml` is the **terminal configuration descriptor** downloaded from the
payment server via the Info frame (see `PAYMENT.md`). The shortened variant
`infoshort.xml` is used when the server indicates a short update is sufficient.

## Purpose

The file is a runtime cache of the server response that configures the terminal:
identity, account, services, operators, limitations, states, and many other
sections. It is not a static deployment artifact.

## Download path

```
TaskGetInfo (periodic task, MainLogic)
  → taskGetInfoHandler
    → Frame(frameType Info | InfoShort) → Formatter.Process()
    → server response is verified and saved:
        FrameType.Info      → info.xml
        FrameType.InfoShort → infoshort.xml
    → parseInfoXml() reads and applies the file
```

The file is overwritten each time the task runs. If the file is missing,
`parseInfoXml` throws (caught by the task handler, retried with a delay).

## Structure

```xml
<info protocol_version="2.5.1" time="...">
  <client id="..." serial="..." name="..." blocked="..." inn="..." kpp="..."
          deal_number="..." deal_date="...">
    <requisites legal_name="..." post_address="..." taxpayer_id_number="..."
                rr_code="..." contact_phone_number="..."/>
  </client>
  <account id="..." serial="..." name="..." allowed_limit="..." balance="..." limit="..."/>
  <point id="..." serial="..." blocked="..." address="..." name="..." type="..." input_date="..."/>
  <states>
    <state id="1000" meaning="Decline"/>
    ...
  </states>
  <operators>
    <operator id="1" name="..."><requisites .../></operator>
  </operators>
  <services>
    <service id="alias" meaning="..." operator_id="..." serial="..." comission="..."
             comtype="..." maxcom="..." mincom="..." fee="..." minvalue="..."
             maxvalue="..." check_necessity="..." recommended_check_timeout="...">
      <commission_limitation max_rate="..." min_rate="..." max_amount="..."
                             min_amount="..." start_date="..." commission_type="..."
                             type="..."/>
    </service>
  </services>
  <!-- further sections: scenarios, steps, config, users, languages,
       bar_codes, help, statuses, modules, procedures, tasks, intricate ... -->
</info>
```

## What parseInfoXml applies

- `/info/@protocol_version` → `Setting.Default.VersionProtocol` (saved on change).
- `/info/point/@id` → `Setting.Default.ClientId` (new terminal identity).
- `/info/point/@serial`, `/info/client/@serial` → `SerialPoint`, `SerialClient`.
- client/point attributes: blocked flags, Address, PointName, Inn, ClientName,
  DealNumber, DateDealNumber, Kpp.
- `/info/client` requisites → `SaveClientRequisite`.
- `/info/account` balance/limit → monitoring messages + block logic.
- `/info/operators/operator` → `SaveOperators` (operators + requisites).
- `/info/services/service` → `SaveServicesFromInfo` (service rows) and
  `SaveServicesLimits` (limitations) for protocol ≥ 2.3.2.
- service attributes: operator_id, external_id, check_necessity,
  recommended_check_timeout, binding/optional/out attributes.

## Notes

- Server replies are signed; the client verifies the signature before saving
  (`Reader.Verify` semantics in the payment client).
- A stale file may be re-used if a later download fails.
- Mock Info responses (test) use `protocol_version="2.5.1"`; see `TESTING.md`.
