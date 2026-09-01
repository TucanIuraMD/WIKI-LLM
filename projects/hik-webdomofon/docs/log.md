# 📄 Анализ документации проекта (лог)

## 📋 Таблица всех Markdown‑файлов
| № | Путь | Предназначение | Актуальность | Дублирует | Рекомендация |
|---|------|----------------|--------------|----------|--------------|
| 1 | **README.md** | "Витрина" проекта, быстрый старт, ссылки на другие доки. | ✅ актуален | частично `docs/00_Project.md` (повторный ввод) | оставить (добавить ссылки на новые master‑документы). |
| 2 | **docs/00_Project.md** | Полный ввод: цель, состав, high‑level обзор. | ✅ актуален | перекрывается с `README.md` (повторный ввод) | оставить (главный "Welcome‑guide"). |
| 3 | **docs/01_Roadmap.md** | План развития, текущий статус этапов. | ✅ актуален | небольшие пересечения с `docs/04_DECISIONS.md` | оставить. |
| 4 | **docs/02_AI_CONTEXT.md** | Правила поведения ИИ‑модели (не используется). | ⛔ устарел | полностью дублирует `docs/README_AI.md` | **удалить** (или переместить в `archive/`). |
| 5 | **docs/03_ARCHITECTURE.md** | Схема компонентов, принципы взаимодействия. | ✅ актуален | частично дублирует `PROJECT_ARCHITECTURE.md` | оставить (добавить кросс‑ссылку). |
| 6 | **docs/04_DECISIONS.md** | Архитектурные/технические решения, причины. | ✅ актуален | частично перекрывается с `docs/03_ARCHITECTURE.md` | оставить (добавить кросс‑ссылку). |
| 7 | **docs/05_PROJECT_STATE.md** | Текущее состояние проекта, реализовано/не реализовано. | ✅ актуален | частично совпадает с `docs/01_Roadmap.md` | оставить. |
| 8 | **docs/06_API.md** | Таблица всех публичных HTTP‑эндпоинтов, описание запросов/ответов. | ✅ актуален | полностью дублирует `API_MAP.md` | оставить (удалить `API_MAP.md`). |
| 9 | **docs/07_TELEGRAM.md** | Описание взаимодействия Bot ↔ Telegram, команды, WebApp. | ✅ актуален | нет дублей | оставить. |
|10| **docs/08_MINI_APP.md** | Описание UI Mini‑App (видеопоток, аудио, двери). | ✅ актуален | части дублируются в `docs/09_VIDEO.md` + `docs/10_AUDIO.md` | **объединить** → `docs/FRONTEND_OVERVIEW.md`. |
|11| **docs/09_VIDEO.md** | Технические детали видеопотока (go2rtc, каналы). | ✅ актуален | частично ↔ `docs/08_MINI_APP.md` | **объединить** → `docs/FRONTEND_OVERVIEW.md`. |
|12| **docs/10_AUDIO.md** | Описание аудио‑потока, кодек µ‑law, VoiceSession. | ✅ актуален | части дублируются в `docs/08_MINI_APP.md` и `RUNTIME_FLOW.md` | **объединить** → `docs/FRONTEND_OVERVIEW.md`. |
|13| **docs/11_DATABASE.md** | Схема SQLite‑БД, таблицы, модели. | ✅ актуален | нет дублей | оставить. |
|14| **docs/12_SECURITY.md** | Меры безопасности (TOKEN, CORS, пр.). | ✅ актуален | нет дублей | оставить. |
|15| **docs/13_DEPLOYMENT.md** | Инструкция по запуску в Docker‑Compose, переменные env. | ✅ актуален | частично дублируется в `README.md` | оставить (можно позже объединить с README). |
|16| **docs/14_TEST_PLAN.md** | План тестирования, типы тестов. | ✅ актуален | небольшие пересечения с `docs/15_BUGS.md` | оставить. |
|17| **docs/15_BUGS.md** | Список известных багов и их статус. | ✅ актуален | частично ↔ `docs/14_TEST_PLAN.md` | оставить. |
|18| **docs/16_CHANGELOG.md** | Хронология изменений проекта. | ✅ актуален | нет дублей | оставить. |
|19| **docs/17_DEVELOPMENT_RULES.md** | Правила разработки, стиль, CI. | ✅ актуален | нет дублей | оставить. |
|20| **docs/README_AI.md** | Инструкция по использованию AI (не применимо). | ⛔ устарел | полностью дублирует `docs/02_AI_CONTEXT.md` | **удалить** (или переместить в `archive/`). |
|21| **PROJECT_ARCHITECTURE.md** | Описание сервисов Docker‑контейнеров и их ролей. | ✅ актуален | частично дублирует `docs/03_ARCHITECTURE.md` | оставить (добавить кросс‑ссылку). |
|22| **HCNETSDK_MAP.md** | Описание функций SDK‑обёртки. | ✅ актуален | сильно пересекается с `SDK_FLOW.md` (который будет удалён) | оставить (полезно для разработчиков, работающих с SDK). |
|23| **REFACTOR_PLAN.md** | План безопасного удаления мёртвого кода. | ✅ актуален | почти полностью повторяется в `DEAD_CODE_REPORT.md` | **удалить** (сохранить в архив). |
|24| **DEAD_CODE_REPORT.md** | Перечень неиспользуемых функций, модулей, скриптов. | ✅ актуален | дублирует `REFACTOR_PLAN.md` | оставить (после выполнения рефакторинга можно будет архивировать). |
|25| **REPORT.md** | Сводный отчёт о текущем состоянии (статистика, скриншоты). | ✅ актуален | частично дублирует `TEST_RESULTS.md` | оставить (удалить `TEST_RESULTS.md`). |
|26| **TEST_RESULTS.md** | Выводы CI‑тестов. | ✅ актуален | дублирует `REPORT.md` | **удалить** (или переместить в архив). |
|27| **TASK_TEMPLATE.md** | Шаблон задачи для проекта. | ✅ актуален | нет дублей | оставить. |
|28| **CONTRIBUTING.md** | Руководство для контрибуторов (форк, PR). | ✅ актуален | нет дублей | оставить. |
|29| **.mimocode/plans/*** | Внутренние планы sub‑агентов MiMoCode (не относятся к пользователю). | ⛔ не требуется новому dev | нет дублирования | **переместить** в `.mimocode/legacy/` (не трогать в репозитории). |

---

## 🗑️ Файлы, предлагаемые к удалению
| Файл | Причина удаления |
|------|-------------------|
| `docs/02_AI_CONTEXT.md` | Проект не использует AI; документ устарел. |
| `docs/README_AI.md` | Дублирует предыдущий AI‑документ, не нужен. |
| `API_MAP.md` | Дублирует `docs/06_API.md`. |
| `SDK_FLOW.md` | Информация полностью перенесена в `RUNTIME_FLOW.md`. |
| `OBJECT_LIFETIME.md` | Перенесено в `RUNTIME_FLOW.md`. |
| `QUEUE_MAP.md` | Перенесено в `RUNTIME_FLOW.md`. |
| `THREAD_MAP.md` | Перенесено в `RUNTIME_FLOW.md`. |
| `FUNCTION_MAP.md` | Дублирует `CALL_GRAPH.md`. |
| `REFACTOR_PLAN.md` | Дублирует `DEAD_CODE_REPORT.md`. |
| `TEST_RESULTS.md` | Дублирует `REPORT.md`. |
| `talk_server.py` (в корне `hik-talk/`) | Не используется в текущей работе проекта. |
| `tools/voice_test.py` | Тестовый скрипт, не часть основной логики. |

---

## 🔀 Файлы, предлагаемые к объединению
| Итоговый документ | Исходные файлы, которые объединяются |
|-------------------|-----------------------------------|
| `docs/FRONTEND_OVERVIEW.md` | `docs/08_MINI_APP.md`, `docs/09_VIDEO.md`, `docs/10_AUDIO.md` |
| `RUNTIME_FLOW.md` (расширенный) | `RUNTIME_FLOW.md`, `OBJECT_LIFETIME.md`, `SDK_FLOW.md`, `QUEUE_MAP.md`, `THREAD_MAP.md` |
| `docs/06_API.md` | `docs/API_MAP.md` |

---

## 📂 Файлы, предлагаемые к перемещению (в архив)
| Текущий путь | Новый путь (архив) |
|--------------|-------------------|
| `docs/02_AI_CONTEXT.md` | `docs/archive/02_AI_CONTEXT.md` |
| `docs/README_AI.md` | `docs/archive/README_AI.md` |
| `docs/API_MAP.md` | `docs/archive/API_MAP.md` |
| `docs/SDK_FLOW.md` | `docs/archive/SDK_FLOW.md` |
| `docs/OBJECT_LIFETIME.md` | `docs/archive/OBJECT_LIFETIME.md` |
| `docs/QUEUE_MAP.md` | `docs/archive/QUEUE_MAP.md` |
| `docs/THREAD_MAP.md` | `docs/archive/THREAD_MAP.md` |
| `docs/FUNCTION_MAP.md` | `docs/archive/FUNCTION_MAP.md` |
| `docs/REFACTOR_PLAN.md` | `docs/archive/REFACTOR_PLAN.md` |
| `docs/DEAD_CODE_REPORT.md` | `docs/archive/DEAD_CODE_REPORT.md` |
| `docs/TEST_RESULTS.md` | `docs/archive/TEST_RESULTS.md` |
| `hik-talk/talk_server.py` | `docs/archive/talk_server.py` |
| `hik-talk/tools/voice_test.py` | `docs/archive/voice_test.py` |

---

## ✨ Новые документы, которые будут созданы
| Новый документ | Содержание / цель |
|----------------|-------------------|
| `docs/FRONTEND_OVERVIEW.md` | Сводный обзор UI Mini‑App, видеопотока и аудиопотока (заменяет 08/09/10). |
| `docs/MODULE_OVERVIEW.md` | Объединённый обзор импортов и внешних зависимостей (заменяет `MODULE_CALLS` + `MODULE_DEPENDENCIES`). |
| `RUNTIME_FLOW.md` (расширенный) | Подробный сценарий выполнения, жизненный цикл объектов, SDK‑взаимодействие, очереди, потоки. |
| `README.md` (новый/обновлённый) | Главная точка входа: краткое описание проекта, быстрый старт, ссылки на master‑документы. |
| `docs/archive/…` | Папка с историческими/устаревшими документами, чтобы их не удалять полностью. |

---

## 🌳 Итоговая структура каталога `docs/`
```
/docs
├─ 00_Project.md                     # главный ввод (keep)
├─ 01_Roadmap.md                     # план развития (keep)
├─ 03_ARCHITECTURE.md                # схема компонентов (keep)
├─ 04_DECISIONS.md                   # технические решения (keep)
├─ 05_PROJECT_STATE.md                # текущее состояние (keep)
├─ 06_API.md                         # единственная таблица API (keep)
├─ 07_TELEGRAM.md                    # Bot ↔ Telegram (keep)
├─ 11_DATABASE.md                    # схема БД (keep)
├─ 12_SECURITY.md                    # меры безопасности (keep)
├─ 13_DEPLOYMENT.md                  # запуск в Docker (keep)
├─ 14_TEST_PLAN.md                   # план тестирования (keep)
├─ 15_BUGS.md                        # известные баги (keep)
├─ 16_CHANGELOG.md                   # журнал изменений (keep)
├─ 17_DEVELOPMENT_RULES.md           # правила разработки (keep)
├─ FRONTEND_OVERVIEW.md               # **new** – объединённый UI‑документ
├─ MODULE_OVERVIEW.md                 # **new** – объединённый обзор модулей
├─ RUNTIME_FLOW.md                    # **updated** – полный runtime‑документ
├─ archive/                           # архив устаревших документов
│   ├─ 02_AI_CONTEXT.md
│   ├─ README_AI.md
│   ├─ API_MAP.md
│   ├─ SDK_FLOW.md
│   ├─ OBJECT_LIFETIME.md
│   ├─ QUEUE_MAP.md
│   ├─ THREAD_MAP.md
│   ├─ FUNCTION_MAP.md
│   ├─ REFACTOR_PLAN.md
│   ├─ DEAD_CODE_REPORT.md
│   └─ TEST_RESULTS.md
└─ (другие справочные файлы, если есть)
```

---

*Все изменения сохраняются в `docs/log.md` (этот файл), чтобы вы могли проверить и при необходимости откорректировать перед выполнением реальных git‑команд.*