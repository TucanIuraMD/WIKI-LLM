# Logic Linux — Technical Knowledge Base (AI-WIKI)

This repository contains the **technical documentation** for the Logic Linux project
(a self-service payment kiosk terminal software, ported from a Windows application to
Linux/Mono). The application source code lives in the companion repository
**Logic.git**; this wiki is the primary technical knowledge base for AI assistants and
developers maintaining the project.

## Repository layout

```
WIKI-LLM/
└── docs/
    ├── README.md          ← this file (navigation)
    ├── LLM_CONTEXT.md     ← main entry point / context for AI engineers
    ├── ARCHITECTURE.md    ← components, data flows, processes
    ├── LINUX_PORT.md      ← Windows → Linux migration summary
    ├── DATABASE.md        ← external MySQL database contract
    ├── DEVICES.md         ← DeviceCenter, CashCode, Citizen PPU 700, serial layer
    ├── NETWORK.md         ← network overview (monitoring + payment)
    ├── PAYMENT.md         ← payment protocol (HTTP), signed frames, Info/InfoShort
    ├── MONITORING.md      ← monitoring protocol (TCP)
    ├── INFO_XML.md        ← info.xml / terminal configuration descriptor
    ├── TESTING.md         ← test executables, mock servers, E2E
    ├── TROUBLESHOOTING.md ← symptoms, causes, solutions
    └── DEPLOYMENT.md      ← production deployment outline
```

## Reading order

1. `LLM_CONTEXT.md` — project identity, rules, status.
2. `ARCHITECTURE.md` — component map.
3. Then the topic-specific documents as needed.

## Security conventions used in this wiki

- Real host names/IPs are replaced by placeholders: `127.0.0.1`, `example.local`,
  `<SERVER_HOST>`, `<MONITORING_SERVER>`, `<PAYMENT_SERVER>`.
- Credentials are replaced by `<DB_USER>`, `<DB_PASSWORD>`, `<TEST_KEY>`.
- Internal development paths and audit materials are **not** published here.
- Raw audit files (`INFO_*_AUDIT.md` etc.) are intentionally **not** part of this wiki.

## Related repositories

- Application repository: `Logic.git` (program, safe examples, minimal docs).
- This wiki: `WIKI-LLM.git` (detailed technical documentation).
