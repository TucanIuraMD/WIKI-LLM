# Payment Protocol

The terminal communicates with a remote payment server via HTTP POST of
**signed binary frames** (SinkLib/Formatter).

## Transport

- **Endpoint**: `http://<PAYMENT_SERVER>:14111` (configured via `PaymentUri`
  setting, optionally through an HTTP proxy).
- **Method**: POST to the base URI, no extra path.
- **Headers**: `Accept-Encoding: deflate`; optionally `eKassir-PointID` and
  `eKassir-Password` (for a specific authentication mode).

## Request format

The HTTP body is a **base64** string (ASCII) concatenating:

```
keyId    16 bytes   = first 16 bytes of the RSA public-key modulus
dataLen   4 bytes   = big-endian Int32
data      dataLen   = UTF-16LE encoded XML
signature remainder = RSA-MD5 signature of `data` (signed with the private key)
```

## Frame types (XML root elements)

| FrameType | root tag | Purpose |
|-----------|----------|---------|
| Process (0) | `<process>` | submit a payment for processing |
| Check (1) | `<check>` | verify account existence |
| Info (2) | `<info>` | download terminal configuration (see `INFO_XML.md`) |
| GetBalance (3) | `<balance>` | query balance |
| InfoShort (6) | `<info>` (with `include_services="False"`) | configuration header only |

## Response format

The server replies with **plain XML** (not signed):

```xml
<response>
  <result state="2000" account="..." id="..." />
</response>
```

The client parses the `result` element:

- `state` — integer status code.
- Other attributes are stored in the payment's properties.

## State codes

| Code | Meaning |
|------|---------|
| 2000 | Accepted |
| 1000 | Rejected / Decline |
| 3000 | Under processing |
| 6000 | Cannot check account |
| 7000 | Account exists |
| 8000 | Account not exists |
| 10000 | Source exists |

## Client implementation

- `IBP.Formatter` (SinkLib.dll) orchestrates the request/response cycle.
- `IBP.Frame` builds the frame XML, reads the result.
- `IBP.Class606` loads RSA keys, `Class607` does the HTTP POST.

## RSA signing

- Private key is loaded from `cashin.keys` (see `DATABASE.md`).
- MD5 hash of the UTF-16LE XML data is signed with RSA-PKCS#1-v1.5.
- The public key's modulus prefix (16 bytes) identifies the key to the server.

## Info / InfoShort

Special frame types that download the terminal configuration descriptor
(`info.xml`). See `INFO_XML.md` for details.

## Test mock

A test mock server (`payment_mock.py`) is available for protocol-level testing
without a real payment server (see `TESTING.md`).