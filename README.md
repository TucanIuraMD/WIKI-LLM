# WIKI-LLM

Obsidian vault, ведущийся как LLM-Wiki: LLM-агент превращает сырые источники
в связанную, поддерживаемую базу знаний. Основа — паттерн LLM Wiki (Karpathy).

## Структура

- `raw/` — неизменяемые источники истины (статьи, web-clippings, pdf, ссылки).
- `wiki/` — обработанные страницы: концепты, саммари, синтезы, source notes.
  Точка входа: `wiki/index.md`. Журнал: `wiki/log.md`.
- `projects/` — проектная документация, синхронизированная через Doc-API
  (см. ниже). Каждый подкаталог — зеркало `*.md` из GitHub-репозитория проекта.
- `AGENTS.md` — единственный schema-файл с правилами для LLM-агента.
- `.obsidian/` — настройки Obsidian.

## Workflow

1. **Knowledge**: новые источники попадают в `raw/` (Obsidian Web Clipper),
   агент ингейстит их в `wiki/` по правилам `AGENTS.md`.
2. **Projects**: команда `project-update` (см. репозиторий `project-update`)
   синхронизирует документацию Git-проектов через Doc-API в `projects/<project>/`:

   ```
   Git-проект → project-update → Doc-API → projects/<project>/
   ```

   Агент может читать эту документацию через AI-Wiki MCP / WebObsidian
   для принятия архитектурных решений и кристаллизации знаний в `wiki/`.

## Требования

- Obsidian (для ручного просмотра/редактирования) или WebObsidian (web).
- Git для версионирования vault.
- Опционально: Doc-API + project-update для синхронизации проектной документации.

## Секреты

- Никаких токенов, паролей и API-ключей в репозитории.
- Локальное состояние Obsidian (workspace, cache) не коммитится (.gitignore).

## Связанные репозитории

- `AI-Wiki-MCP` — MCP-сервер для чтения/записи vault через WebObsidian.
- `WebObsidian` — веб-интерфейс + Agent API для vault.
- `Doc-API` — синхронизация проектной документации.
- `project-update` — универсальная команда `project-update`.
