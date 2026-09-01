# WIKI-LLM — Architecture & Operations Manual

> **Главный документ инфраструктуры.**
> Цель: любой человек или AI, впервые попавший в систему, открывает этот файл
> и полностью понимает, что такое WIKI-LLM, как она устроена и как ей пользоваться.
>
> Этот документ описывает **реальную текущую инфраструктуру** (проверено по
> конфигурациям, Git-репозиториям и живым сервисам на 2026-08-31).

---

## Navigation

Карта связей этого документа и vault:

- [[WIKI-LLM]] — главный архитектурный документ (этот файл)
- [[AGENTS]] — правила для LLM-агента (schema)
- [[README]] — общее описание vault
- [[wiki/index]] — карта базы знаний (центральный индекс)
- [[wiki/LLM Wiki]] — паттерн LLM Wiki
- [[wiki/LLM Wiki Workflow]] — операции ingest / query / lint / crystallization
- [[wiki/LLM Wiki Lifecycle]] — confidence / freshness / supersession
- [[projects/index]] — навигация по проектам

## Projects

- [[projects/NetMap/README]] — NetMap (сетевая инфраструктура)
- [[projects/WebOllama/README]] — WebOllama (Web-панель управления Ollama)
- [[projects/ReadmeMD/README]] — ReadmeMD (просмотрщик Obsidian vault)
- [[projects/dev-agent/README]] — dev-agent (AI-агент разработки)
- [[projects/hik-webdomofon/README]] — hik-webdomofon (Web-домофон Hikvision)


## Quick Reference (30 секунд)

| Компонент | Хост | Порт | Роль | Точка входа |
|---|---|---|---|---|
| WIKI-LLM (vault) | 22, 111 | — | Единая база знаний (Obsidian vault) | `/opt/ai-stack/wiki-llm/` |
| WebObsidian | 22, 111 | 8787 | HTTP API + Web UI для vault | `http://<host>:8787` |
| AI-Wiki-MCP | 22, 111 | 8790 | MCP-сервер (Streamable HTTP) для LLM | `http://<host>:8790/mcp` |
| MCPO | 22, 111 | 8791 | MCP → OpenAPI-адаптер для OpenWebUI | `http://<host>:8791/openapi.json` |
| Doc-API | 22, 111 | 12121 | Синхронизация проектной документации | `http://<host>:12121/health` |
| project-update | 22, 111 | — | CLI-команда для запуска синхронизации | `/usr/local/bin/project-update` |
| OpenWebUI | 22, 111 | 3000 | Пользовательский AI-интерфейс | `http://<host>:3000` |
| Ollama | **22** | 11434 | Локальный LLM runtime | `http://192.168.80.22:11434` |
| Qdrant | 22, 111 | 6333 | Векторная БД для RAG OpenWebUI | `http://<host>:6333` |
| WebOllama | 22, 111 | 8080 | Web-панель управления Ollama | `http://<host>:8080` |

> **Хосты:** `192.168.80.22` = PRODUCTION/CENTRAL, `192.168.80.111` = TEST POLYGON.

---

## 1. Что такое WIKI-LLM

WIKI-LLM — это **единая центральная память для всех AI** в локальной
инфраструктуре. Она реализована как **Obsidian vault** (набор Markdown-файлов),
обслуживаемый **WebObsidian** и доступный AI через **AI-Wiki-MCP**.

### Зачем она нужна

- Все AI-инструменты (OpenWebUI, Claude, Hermes/DeepSeek Harness) используют
  **одну общую базу знаний**, а не свои изолированные памяти.
- Знания накапливаются структурированно (LLM-Wiki-паттерн, см. `wiki/`).
- Документация проектов синхронизируется из Git автоматически.
- Человек и AI работают с одним источником знаний.

### Главный принцип (не смешивать роли)

| Источник | Что хранит |
|---|---|
| **Git** | source of truth для **кода** проектов |
| **WIKI-LLM** | source of truth для **знаний, решений, состояния, контекста** проектов |

- AI-Wiki-MCP = стандартный интерфейс AI → WIKI.
- WebObsidian = хранилище + HTTP-интерфейс WIKI.
- Doc-API = backend синхронизации проектной документации.
- project-update = команда запуска синхронизации проекта → WIKI.
- MCPO = адаптер MCP → OpenAPI для OpenWebUI.
- OpenWebUI = пользовательский AI-интерфейс.
- Ollama = локальный LLM runtime.
- Qdrant = vector/RAG инфраструктура OpenWebUI (не хранилище WIKI).

---

## 2. Серверы

### 192.168.80.22 — PRODUCTION / CENTRAL

Центральная AI-машина. Здесь живут production-сервисы:
OpenWebUI, Ollama, Qdrant, WebObsidian, AI-Wiki-MCP, MCPO, Doc-API, WebOllama, WIKI-LLM, project-update.

### 192.168.80.111 — TEST POLYGON

Тестовый полигон — полная копия стека для разработки и проверки изменений
перед переносом на 22. Тут тоже развёрнуты: OpenWebUI, Qdrant, WebObsidian,
AI-Wiki-MCP, MCPO, Doc-API, WebOllama, WIKI-LLM, project-update.

> **Ollama на 111 отсутствует** (на 111 нет порта 11434). Ollama живёт только на 22.

**Правило:** не путать хосты. Тесты — на 111, production — на 22.

---

## 3. Компоненты и их устройство

### 3.1 WIKI-LLM (vault)

- **Путь на диске:** `/opt/ai-stack/wiki-llm/`
- **Git:** репозиторий `TucanIuraMD/WIKI-LLM` (staging: `/opt/ai-stack/repos-staging/WIKI-LLM/`, remote `git@github.com:TucanIuraMD/WIKI-LLM.git`)

**Структура vault** (проверено фактически):

```
/opt/ai-stack/wiki-llm/
├── AGENTS.md          # правила для LLM-агента (schema-файл)
├── README.md          # описание vault
├── .obsidian/         # настройки Obsidian (app, appearance, core-plugins, graph)
├── Daily/             # ежедневные заметки (например 2026-08-31.md)
├── raw/               # НЕИЗМЕНЯЕМЫЕ источники истины (статьи, web-clippings)
├── wiki/              # обработанные знания (концепты, саммари, source notes)
│   ├── index.md       # карта базы знаний / точка входа
│   ├── log.md         # журнал изменений
│   ├── Source Notes.md# реестр ingested источников
│   └── ...            # тематические страницы (LLM Wiki, Zettelkasten, PARA и др.)
└── projects/          # документация проектов, синхронизируемая через Doc-API
    └── WebOllama/     # (пример: зеркало *.md из репозитория проекта)
```

**Роль каталогов (по фактическому содержимому):**

- `raw/` — неизменяемые первоисточники; никогда не редактировать/удалять без
  явного разрешения пользователя (правило из `AGENTS.md`).
- `wiki/` — производные страницы знаний: саммари, концепты, синтезы, реестры.
- `projects/<project>/` — **зеркало** `*.md`-файлов Git-репозитория проекта,
  создаваемое Doc-API. Это документация проекта внутри WIKI.
- `AGENTS.md` — единый schema-файл правил для LLM-агента (workflow ingest /
  query / lint, правила confidence, freshness, конфликтов).
- `README.md` — краткое описание vault.
- `Daily/` — ежедневные заметки.
- `.obsidian/` — настройки Obsidian (общие, без секретов).

**Текущий Git-статус vault:** чистый; remote origin не настроен в самом vault
(источник — `repos-staging/WIKI-LLM`). Это зафиксировано в разделе «Противоречия».

### 3.2 WebObsidian

- **Роль:** HTTP API + Web UI для vault.
- **Хост/порт:** 22, 111 → **8787**.
- **Запуск:** Docker-контейнер `webobsidian` (образ `webobsidian:latest`, сборка из `/opt/ai-stack/webobsidian`).
- **Репозиторий:** `TucanIuraMD/WebObsidian`.
- **Конфигурация:** `/opt/ai-stack/webobsidian/.env`:

```
VAULT_HOST_PATH=/opt/ai-stack/wiki-llm   # какой vault обслуживает
HTTP_BIND=0.0.0.0
HTTP_PORT=8787
WEBOBSIDIAN_PASSWORD=<secret>            # пароль администратора (не в Git)
WEBOBSIDIAN_WATCH=auto
TRUST_PROXY=true
```

- **Как работает:** монтирует vault (`VAULT_HOST_PATH`) в контейнер, читает
  Markdown-файлы, отдаёт их через Web UI (`:8787`) и через **Agent API** (`/api/v1/*`).

**Agent API (для AI):**

| Метод | Путь | Scope | Описание |
|---|---|---|---|
| GET | `/api/v1/health` | — | Health check |
| GET | `/api/v1/notes` | read | Список заметок |
| GET | `/api/v1/notes/*` | read | Чтение заметки |
| PUT | `/api/v1/notes/*` | write | Создать/обновить заметку |
| PATCH | `/api/v1/notes/*` | write | Дописать заметку |
| DELETE | `/api/v1/notes/*` | write | Удалить (в `.trash`) |
| GET | `/api/v1/search` | search | Полнотекстовый поиск |
| GET | `/api/v1/backlinks` | read | Обратные ссылки |
| GET | `/api/v1/tags` | read | Список тегов |

**Аутентификация Agent API:** заголовок `X-API-Key: <AI_WIKI_API_KEY>`.

### 3.3 AI-Wiki-MCP

- **Роль:** MCP-сервер — **стандартный интерфейс AI → WIKI**.
- **Хост/порт:** 22, 111 → **8790**, endpoint `/mcp`.
- **Запуск:** Docker-контейнер `ai-wiki-mcp` (образ `ai-stack-ai-wiki-mcp`, сборка из `/opt/ai-stack/ai-wiki-mcp`).
- **Репозиторий:** `TucanIuraMD/AI-Wiki-MCP`.
- **Конфигурация:** `/opt/ai-stack/ai-wiki-mcp/.env`:

```
AI_WIKI_URL=http://webobsidian:8787   # внутри docker-сети ai-network
AI_WIKI_API_KEY=<secret>             # ключ для WebObsidian Agent API (не в Git)
```

- **Протокол:** **MCP Streamable HTTP** (JSON-RPC 2.0 поверх HTTP, ответ —
  SSE `text/event-stream`). Клиент шлёт `POST /mcp` с заголовками
  `Content-Type: application/json`, `Accept: application/json, text/event-stream`.

**MCP-инструменты (4):**

| Инструмент | Назначение | Read-only |
|---|---|---|
| `ai_wiki_search` | Полнотекстовый поиск по WIKI (по заголовкам, тегам, телу) | ✅ да |
| `ai_wiki_read` | Чтение одного заметки по vault-relative пути (`*.md`) | ✅ да |
| `ai_wiki_write` | Создать/обновить заметку (PUT upsert) | ❌ пишет |
| `ai_wiki_delete` | Удалить заметку (WebObsidian переносит в `.trash`) | ❌ удаляет |

**Поток:**
```
AI → AI-Wiki-MCP (:8790) → WebObsidian Agent API (:8787) → WIKI-LLM (files)
```

AI-Wiki-MCP **не трогает файлы напрямую** — все операции через HTTP API WebObsidian.

### 3.4 MCPO (MCP → OpenAPI proxy)

- **Роль:** адаптер, который выставляет MCP-инструменты как **OpenAPI/REST**
  для OpenWebUI (OpenWebUI не умеет MCP-протокол, но умеет OpenAPI).
- **Хост/порт:** 22, 111 → **8791**.
- **Запуск:** Docker-контейнер `mcpo-ai-wiki` (образ `ghcr.io/open-webui/mcpo:main`).
- **Репозиторий:** официальный `open-webui/mcpo`; наш deployment —
  `/opt/ai-stack/mcpo-ai-wiki/`.
- **Конфигурация:** `/opt/ai-stack/mcpo-ai-wiki/.env`:

```
MCPO_PORT=8791
MCPO_API_KEY=<secret>                                   # ключ OpenAPI (не в Git)
AI_WIKI_MCP_URL=http://192.168.80.111:8790/mcp          # на 111
AI_WIKI_MCP_URL=http://192.168.80.22:8790/mcp           # на 22
```

- **Как запущен:** `mcpo --port 8791 --api-key <key> --server-type streamable-http -- <AI_WIKI_MCP_URL>`
- **Почему MCPO существует:** OpenWebUI получает AI-Wiki **через OpenAPI**, а
  не через MCP напрямую. MCPO — прослойка.

**OpenAPI-эндпоинты MCPO:**

| Метод | Путь | Назначение |
|---|---|---|
| GET | `/openapi.json` | OpenAPI-схема (все инструменты) |
| GET | `/docs` | Swagger UI |
| POST | `/ai_wiki_search` | Поиск по WIKI |
| POST | `/ai_wiki_read` | Чтение заметки |
| POST | `/ai_wiki_write` | Запись заметки |
| POST | `/ai_wiki_delete` | Удаление заметки |

**Аутентификация MCPO:** Bearer-токен — заголовок `Authorization: Bearer <MCPO_API_KEY>`.

**Поток:**
```
OpenWebUI → MCPO (:8791, OpenAPI) → AI-Wiki-MCP (:8790, MCP) → WebObsidian (:8787) → WIKI-LLM
```

### 3.5 Doc-API

- **Роль:** backend синхронизации проектной документации из Git в WIKI.
- **Хост/порт:** 22, 111 → **12121**.
- **Запуск:**
  - на 22: systemd-юнит `doc-api.service` (см. `Doc-API/doc-api.service`), env-файл `/etc/doc-api.env`;
  - на 111: bare-metal процесс `python3 docapi.py` (без systemd, без Docker).
- **Код:** `/opt/ai-stack/doc-api/docapi.py`, конфиг `/opt/ai-stack/doc-api/config.json`.
- **Репозиторий:** `TucanIuraMD/Doc-API` (staging: `/opt/ai-stack/repos-staging/Doc-API/`).

**HTTP endpoints Doc-API:**

| Метод | Путь | Auth | Описание |
|---|---|---|---|
| GET | `/health` | нет | liveness check |
| GET | `/projects` | X-API-Token | whitelist + локальное состояние проектов |
| POST | `/update/<project>` | X-API-Token | git fetch/reset + mirror *.md в vault |

**Конфигурация (`config.json` / env):**

```
host        0.0.0.0
port        12121
projects_dir /root/projects   # на 111; на 22 — /opt/projects
vault_dir   /opt/ai-stack/wiki-llm/projects
branch      main
whitelist   [WebOllama]       # на 111; на 22 — больше проектов
token       <secret или пусто>
```

**Workflow (что делает Doc-API при POST /update/<project>):**
1. Валидирует имя проекта против whitelist + regex (`^[A-Za-z0-9][A-Za-z0-9._-]{0,127}$`).
2. `git fetch` + `git reset --hard origin/<branch>` в локальном clone проекта
   (`projects_dir/<project>`), GitHub = source of truth.
3. Зеркалирует **все `*.md`** файлы проекта в `vault_dir/<project>/`,
   исключая `.git`, `venv`, `node_modules`, `__pycache__`, `.npm-cache` и т.п.
4. Возвращает `{"status":"ok","commit":..., "md_copied":<n>}`.

**Безопасность:** нет shell-выполнения (git-аргументы фиксированы, без shell);
имя проекта — только из URL и только из whitelist; state-changing endpoints
требуют API-токена (`X-API-Token` или `?token=`).

### 3.6 project-update

- **Роль:** CLI-команда, которая запускает синхронизацию текущего Git-проекта
  через Doc-API в WIKI.
- **Где:** `/usr/local/bin/project-update` (симлинк/копия скрипта из
  `/opt/ai-stack/scripts/project-update`), конфиг `/opt/ai-stack/scripts/project-update.conf`.
- **Репозиторий:** `TucanIuraMD/project-update` (staging: `/opt/ai-stack/repos-staging/project-update/`).

**Конфиг (`project-update.conf`):**

```
DOCAPI_HOST=127.0.0.1
DOCAPI_PORT=12121
DOCAPI_TOKEN=<secret>
```

**Как работает project-update (реальный код):**
1. Определяет имя проекта из **Git remote origin** (последний сегмент пути
   `Owner/Name.git`), НЕ из имени каталога.
2. Проверяет, что Doc-API доступен (`GET /projects`) и проект в whitelist.
3. Вызывает `POST /update/<project>`.
4. Проверяет HTTP 200 + `status=ok`.
5. Проверяет, что все tracked `*.md` файлы появились в `projects/<project>/`.
6. Проверяет, что `raw/`, `wiki/`, `AGENTS.md` WIKI **не изменились**.

**Команда НЕ делает** `git pull/reset/commit/push` и не трогает код проекта —
только синхронизацию документации.

### 3.7 OpenWebUI

- **Роль:** пользовательский AI-интерфейс (чат, управление моделями, RAG).
- **Хост/порт:** 22, 111 → **3000**.
- **Запуск:** Docker-контейнер `open-webui` (образ `ghcr.io/open-webui/open-webui:main`).
- **Конфигурация:** в общем `/opt/ai-stack/docker-compose.yml`:

```
OLLAMA_BASE_URL: http://192.168.80.22:11434   # Ollama живёт на 22
VECTOR_DB: qdrant
QDRANT_URI: http://qdrant:6333
RAG_EMBEDDING_ENGINE: ollama
RAG_EMBEDDING_MODEL: bge-m3:latest
ENABLE_RAG_HYBRID_SEARCH: true
```

- **AI-Wiki через OpenAPI:** OpenWebUI подключается к MCPO как к
  OpenAPI-серверу (`http://<host>:8791/openapi.json`, ключ `<MCPO_API_KEY>`),
  после чего в чате/моделях доступны инструменты `ai_wiki_search/read/write/delete`.

### 3.8 Ollama

- **Роль:** локальный LLM runtime.
- **Хост/порт:** **только 22** → **11434** (на 111 нет).
- **API:** OpenAI-совместимый (`http://192.168.80.22:11434`).
- **Модели:** Qwen3.5, qwen3-vl, deepseek, omnicoder, gemma4, gpt-oss, bge-m3 (embedding) и др.
- **Управление:** через WebOllama (:8080) и Ollama Console.

### 3.9 Qdrant

- **Роль:** векторная БД **для RAG OpenWebUI** (коллекции `open-webui_*`).
- **Хост/порт:** 22, 111 → **6333** (REST) / **6334** (gRPC).
- **ВНИМАНИЕ:** WIKI-LLM НЕ хранится в Qdrant. WIKI — это Markdown-файлы
  vault, обслуживаемые WebObsidian. Qdrant — отдельная RAG-инфраструктура OpenWebUI.

### 3.10 WebOllama

- **Роль:** Web-панель управления Ollama (мониторинг GPU, модели, Jobs, LLM API).
- **Хост/порт:** 22, 111 → **8080**.
- **Запуск:** bare-metal Python-процесс (systemd-юнит `ollama-web.service`).
- **Репозиторий проекта:** `TucanIuraMD/WebOllama`, clone в `/root/projects/WebOllama`.

---

## 4. Data Flows (диаграммы)

### 4.1 Общая архитектура

```
                             192.168.80.22 (CENTRAL)
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   ┌──────────┐    ┌───────────┐    ┌──────────────┐     ┌───────────────┐  │
│   │ OpenWebUI│    │ WebOllama │    │   MCPO       │     │  AI-Wiki-MCP  │  │
│   │  :3000   │    │  :8080    │    │  :8791       │     │  :8790        │  │
│   └────┬─────┘    └─────┬─────┘    └─────┬────────┘     └──────┬────────┘  │
│        │                │                │                     │           │
│        │ OpenAI API     │ NVML/Ollama    │ OpenAPI → MCP       │ MCP→HTTP   │
│        ▼                ▼                ▼                     ▼           │
│   ┌──────────┐    ┌──────────┐    ┌──────────────┐     ┌───────────────┐  │
│   │  Ollama  │    │ Qdrant   │    │  WebObsidian │◄────│  WIKI-LLM     │  │
│   │  :11434  │    │  :6333   │    │  :8787       │     │  (vault files)│  │
│   └──────────┘    └──────────┘    └──────┬───────┘     └──────▲────────┘  │
│        ▲                                │                      │           │
│        └──── RAG embeddings ────────────┘         projects/    │ mirror    │
│                                            Doc-API :12121 ──────┘          │
│                                            project-update (CLI)            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 OpenWebUI → AI-Wiki

```
Пользователь → OpenWebUI :3000
                  │  OpenAPI (POST /ai_wiki_search)
                  ▼
              MCPO :8791
                  │  MCP Streamable HTTP (tools/call)
                  ▼
          AI-Wiki-MCP :8790
                  │  HTTP Agent API (GET /api/v1/search)
                  ▼
          WebObsidian :8787
                  │  read filesystem
                  ▼
            WIKI-LLM (wiki/*, projects/*)
```

### 4.3 Claude → AI-Wiki

```
Claude (Desktop / Code)
   │  MCP client, Streamable HTTP transport
   ▼
AI-Wiki-MCP :8790  (http://192.168.80.22:8790/mcp)
   │  HTTP Agent API
   ▼
WebObsidian :8787 → WIKI-LLM
```

### 4.4 Hermes / DeepSeek Harness → AI-Wiki

```
DeepSeek Harness (Hermes)
   │  1) mcp-client plugin → MCP Streamable HTTP
   │  2) skill "ai-wiki" (curl-инструкции)
   ▼
AI-Wiki-MCP :8790 → WebObsidian :8787 → WIKI-LLM
```

### 4.5 project-update → WIKI

```
Developer / AI (в Git-репозитории проекта, напр. /root/projects/WebOllama)
   │  git commit + git push → GitHub
   │  project-update
   ▼
Doc-API :12121
   │  1. git fetch/reset в локальном clone
   │  2. mirror *.md → vault projects/<project>/
   ▼
WIKI-LLM/projects/<project>/
```

### 4.6 Git → WIKI

```
GitHub (source of truth для кода)
   │  push
   ▼
/root/projects/<project>/   (или /opt/projects на 22)
   │  project-update
   ▼
Doc-API → WIKI-LLM/projects/<project>/
   │  читается WebObsidian
   ▼
AI-клиенты (MCP / OpenAPI)
```

### 4.7 Пользователь → OpenWebUI → AI-Wiki

```
Пользователь (браузер)
   │
   ▼
OpenWebUI :3000 (чат; модель может вызвать инструмент ai_wiki_search/read)
   │
   ▼
MCPO :8791 → AI-Wiki-MCP :8790 → WebObsidian :8787 → WIKI-LLM
   │
   ▼
Ответ пользователю (с цитатами из WIKI)
```

---

## 5. AI-клиенты (кто реально имеет доступ к WIKI)

| Клиент | Протокол | Endpoint | Инструменты | Где настроено |
|---|---|---|---|---|
| OpenWebUI | OpenAPI (через MCPO) | `http://<host>:8791/openapi.json` | ai_wiki_search/read/write/delete | Admin → OpenAPI сервер |
| Claude | MCP Streamable HTTP | `http://192.168.80.22:8790/mcp` | ai_wiki_search/read/write/delete | `claude mcp add --transport http ai-wiki ...` (см. сек. «Claude») |
| Hermes / DeepSeek Harness | MCP Streamable HTTP + skill | `http://192.168.80.22:8790/mcp` | `mcp__ai-wiki__ai_wiki_*` + curl | `/root/.dsh/profiles/web/cordis.patch.yml`, skill `/root/.dsh/skills/ai-wiki/SKILL.md` |

> Проверено: на 111 реально подключены OpenWebUI (через OpenAPI-вызовы к MCPO)
> и DeepSeek Harness (mcp-client + skill). Конфигурация Claude описана ниже —
> это целевой сценарий на 22 (клиентские конфиги не лежат на сервере).

---

## 6. Security

### Секреты (никогда не публиковать, не коммитить)

| Секрет | Где | Как передаётся |
|---|---|---|
| `<WEBOBSIDIAN_PASSWORD>` | `/opt/ai-stack/webobsidian/.env` | логин в Web UI |
| `<AI_WIKI_API_KEY>` | `/opt/ai-stack/ai-wiki-mcp/.env`, `/root/.dsh/.env` | заголовок `X-API-Key` |
| `<MCPO_API_KEY>` | `/opt/ai-stack/mcpo-ai-wiki/.env` | заголовок `Authorization: Bearer` |
| `<DOCAPI_TOKEN>` | `/opt/ai-stack/doc-api/config.json` / `/etc/doc-api.env` / `project-update.conf` | заголовок `X-API-Token` |
| `BRAVE_SEARCH_API_KEY` | `/opt/ai-stack/docker-compose.yml` (OpenWebUI env) | внутренний |

- Все `.env` файлы в `.gitignore` соответствующих репозиториев.
- В этом документе ключи заменены плейсхолдерами `<...>`.

### Правила

- Ключи не коммитить в Git, не класть в WIKI, не логировать.
- WebObsidian Agent API: требует `X-API-Key` для `/api/v1/*`.
- MCPO: требует Bearer-токен; healthcheck проходит с ключом.
- Doc-API: `/projects` и `/update/*` требуют токен (на 22 `DOCAPI_REQUIRE_TOKEN=true`).
- Доступ к портам: сервисы опубликованы на `0.0.0.0` — защищены паролями/ключами приложения.

---

## 7. Port Table

| Service | Host | Port | Protocol | Purpose | Auth | Depends on |
|---|---|---|---|---|---|---|
| OpenWebUI | 22, 111 | 3000 | HTTP | AI chat UI + RAG | login form | Ollama, Qdrant |
| WebOllama | 22, 111 | 8080 | HTTP | Ollama management panel | login | Ollama |
| WebObsidian | 22, 111 | 8787 | HTTP | vault API + Web UI | X-API-Key / password | WIKI-LLM |
| AI-Wiki-MCP | 22, 111 | 8790 | MCP Streamable HTTP | AI → WIKI | — (client-side) | WebObsidian |
| MCPO | 22, 111 | 8791 | HTTP (OpenAPI) | MCP→OpenAPI adapter | Bearer token | AI-Wiki-MCP |
| Doc-API | 22, 111 | 12121 | HTTP | project doc sync | X-API-Token | Git, WIKI-LLM |
| Qdrant | 22, 111 | 6333/6334 | HTTP/gRPC | vector DB (OpenWebUI RAG) | — | — |
| Ollama | **22** | 11434 | HTTP (OpenAI) | LLM runtime | — | — |
| project-update | 22, 111 | — | CLI | sync command | token from conf | Doc-API |

---

## 8. How to use

### 8.1 «Я впервые пришёл в WIKI-LLM. Что делать?»

1. **Узнай, где какой сервер:**
   - 22 = production, 111 = тест. Работай с тем, что нужно.
2. **Открой WIKI в браузере:** `http://<host>:8787` (WebObsidian Web UI).
3. **Прочитай правила:** `/opt/ai-stack/wiki-llm/AGENTS.md`.
4. **Посмотри карту знаний:** `/opt/ai-stack/wiki-llm/wiki/index.md`.
5. **Проверь сервисы** (см. раздел «Troubleshooting» — команды).
6. **Настрой свой AI-клиент** (см. раздел «AI-клиенты»).

### 8.2 «Я AI. Как мне пользоваться WIKI?»

1. **search** → найти, что уже есть: `ai_wiki_search("<тема>")`.
2. **read** → прочитать релевантные заметки: `ai_wiki_read("<path>.md")`.
3. **Работа с кодом через Git** (код — не в WIKI): в репозитории проекта
   (`/root/projects/<project>`) — править, `git commit`, `git push`.
4. **Синхронизировать документацию:** из корня проекта выполнить `project-update`.
5. **Проверить WIKI:** `ai_wiki_search("<project>")` / прочитать `projects/<project>/`.
6. **Записать знания** (только с разрешения пользователя): `ai_wiki_write`.

---

## 9. Troubleshooting

### OpenWebUI

```bash
curl -s http://127.0.0.1:3000/api/config          # 200 {status:true,...}
curl -s http://127.0.0.1:3000/api/version         # {"version":"0.11.1"}
docker logs open-webui --tail 50
```
Типичные ошибки: Ollama/Qdrant недоступны → проверить `OLLAMA_BASE_URL`, `QDRANT_URI`.

### MCPO

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8791/openapi.json   # 401 без ключа
curl -s -H "Authorization: Bearer <MCPO_API_KEY>" http://127.0.0.1:8791/openapi.json
docker logs mcpo-ai-wiki --tail 50
bash /opt/ai-stack/mcpo-ai-wiki/health.sh
```
Типичные ошибки: `AI_WIKI_MCP_URL` недоступен → проверить, что AI-Wiki-MCP жив.

### AI-Wiki-MCP

```bash
# MCP initialize
curl -s -X POST http://127.0.0.1:8790/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  --data '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"diag","version":"1"}}}'
docker logs ai-wiki-mcp --tail 50
```
Ожидаемо: `serverInfo: {name:"AI-Wiki", version:"1.29.1"}`.

### WebObsidian

```bash
curl -s http://127.0.0.1:8787/api/v1/health        # {"ok":true,...}
curl -s -H "X-API-Key: <AI_WIKI_API_KEY>" "http://127.0.0.1:8787/api/v1/search?q=test"
docker logs webobsidian --tail 50
```
Ошибки: 401 → неверный ключ; 404 → заметки нет; vault не найден → проверить `VAULT_HOST_PATH`.

### Doc-API

```bash
curl -s http://127.0.0.1:12121/health              # {"status":"ok","service":"doc-api"}
curl -s -H "X-API-Token: <DOCAPI_TOKEN>" http://127.0.0.1:12121/projects
# на 22:
systemctl status doc-api; journalctl -u doc-api --no-pager -n 30
```
Ошибки: 401 → токен; проект не в whitelist → 400/403; git-сбой → смотреть ответ `/update`.

### project-update

```bash
project-update          # из корня Git-проекта
```
Exit-коды (из скрипта): 2 — не git-репо; 3 — нет remote origin;
4 — не извлёк имя; 5 — Doc-API недоступен; 6 — проект не в whitelist;
7 — HTTP ошибка; 8 — status != ok; 9 — файлы не синхронизированы; 10 — нет конфига.

### WIKI-LLM

```bash
ls /opt/ai-stack/wiki-llm/{AGENTS.md,wiki,raw,projects}
git -C /opt/ai-stack/wiki-llm status
find /opt/ai-stack/wiki-llm -name '*.md' | wc -l
```

---

## 10. Rules for AI

**AI должен:**
- сначала **искать** существующие знания (`ai_wiki_search`), потом читать (`ai_wiki_read`);
- не дублировать информацию — дополнять существующие страницы;
- работать с **кодом через Git**, не считать WIKI источником истины для кода;
- после изменения проекта запускать `project-update` для синхронизации документации;
- не писать секреты; не удалять документы без необходимости (delete → `.trash`).

**READ vs WRITE:**
- **READ** (search, read) — всегда разрешено, это основной режим.
- **WRITE** (write, delete) — только по явному запросу пользователя; перед
  записью прочитать текущую версию, не перезаписывать молча, не создавать дубликаты.

---

## 11. Найденные противоречия (требуют внимания)

1. **Путь к vault в этом документе vs фактический.** Файл лежит в
   `/opt/ai-stack/WIKI-LLM/WIKI-LLM.md`, но сам vault — `/opt/ai-stack/wiki-llm`
   (строчные буквы). Linux чувствителен к регистру — это **разные каталоги**.
   Нужно решить, где будет канонический путь.
2. **`repos-staging` vs рабочие каталоги.** Есть `/opt/ai-stack/repos-staging/*`
   (клоны GitHub) и рабочие `/opt/ai-stack/{ai-wiki-mcp,webobsidian,doc-api,...}`.
   Рабочие каталоги могут не иметь Git-remote (проверено: `wiki-llm` и
   `webobsidian` — remote отсутствует). Нужна чёткая схема «где Git-источник».
3. **`/root/projects` vs `/opt/projects`.** Doc-API на 111 сконфигурирован на
   `/root/projects`, а `docapi.py`/`doc-api.service` по умолчанию используют
   `/opt/projects`. На 22 должен быть `/opt/projects` — проверить фактически.
4. **Whitelist Doc-API.** На 111 в `config.json` whitelist = `["WebOllama"]`,
   но проект `install.sh`/ожидание предполагает больше проектов
   (NetMap, hik-webdomofon, dev-agent, ReadmeMD, ia-agent). На 22 — уточнить.
5. **Ollama только на 22.** На 111 Ollama отсутствует, но OpenWebUI на 111
   указывает `OLLAMA_BASE_URL=http://192.168.80.22:11434` — тестовый UI на 111
   ходит за LLM на production-хост. Задокументировано как есть.
6. **Claude-конфигурация** — описана как целевой сценарий; файлы конфигурации
   Claude на сервере не обнаружены (клиентская настройка).

---

## 12. Краткий словарь

| Термин | Значение |
|---|---|
| vault | Obsidian-хранилище Markdown-файлов = WIKI-LLM |
| MCP | Model Context Protocol — стандарт подключения LLM к инструментам |
| Streamable HTTP | транспорт MCP поверх HTTP (JSON-RPC + SSE) |
| OpenAPI | REST-схема, понятная OpenWebUI и многим другим |
| RAG | Retrieval-Augmented Generation (в OpenWebUI через Qdrant) |
| source of truth | источник истины (Git — для кода, WIKI — для знаний) |

---

*Конец документа. Создан на основе фактического исследования инфраструктуры
2026-08-31: `/opt/ai-stack`, Docker-контейнеры, конфигурации, Git-репозитории,
project-update, Doc-API, AI-Wiki-MCP, WebObsidian, WIKI-LLM, MCPO, OpenWebUI,
Ollama, Qdrant.*
