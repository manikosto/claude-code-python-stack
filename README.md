# Python Stack for Claude Code

**[English](#english)** | **[Русский](#russian)**

---

<a id="english"></a>

## What Is This?

A **ready-to-use toolkit** that supercharges Claude Code for Python development and QA automation. Once you drop it into your project, Claude Code instantly becomes a senior Python engineer who follows your team's exact standards.

**Without this toolkit** — Claude Code is a smart generalist.
**With this toolkit** — Claude Code writes Pythonic code, designs architectures, generates OOP test suites with Allure, reviews for OWASP vulnerabilities, and follows your exact conventions.

## How Claude Code Configuration Works

When you run Claude Code in a project, it reads special files from a `.claude/` directory:

```
your-project/
├── CLAUDE.md              # Main instructions — "the brain"
├── .claude/
│   ├── rules/             # Always-on guidelines (activate by file type)
│   ├── skills/            # Deep knowledge (activated when context matches)
│   ├── agents/            # Specialized sub-agents (reviewers, planners, etc.)
│   ├── commands/          # Slash commands you type (/plan, /tdd, /verify)
│   └── hooks.json         # Automatic actions (format on save, type check)
└── src/
```

This toolkit provides all of these components, pre-configured for Python.

---

## Components Explained

### CLAUDE.md — The Brain

The **single most important file**. Sits in your project root. Claude reads it at the start of every conversation and follows it as law.

Our CLAUDE.md includes:
- Python conventions (PEP 8, type hints, black/ruff/mypy)
- Strict OOP test rules (classes, service layers, Allure — mandatory)
- Framework rules (FastAPI, Django, SQLAlchemy, Celery)
- Database rules (PostgreSQL types, indexes, N+1 prevention)
- Security rules (no eval, no hardcoded secrets, bandit)
- All available commands and agents listed

**Think of it as**: an onboarding doc for a new team member. Everything they must always follow.

---

### Rules — Always-On Guidelines

**Location**: `rules/` → copy to `.claude/rules/`

Markdown files that **auto-activate by file pattern**. Edit a `.py` file — Python rules kick in automatically.

```yaml
---
paths:
  - "**/*.py"
---
# Python Coding Style
- Follow PEP 8
- Type annotations on all public functions
- black for formatting, ruff for linting
```

| File | What It Enforces |
|------|-----------------|
| `coding-style.md` | PEP 8, type hints, immutability, formatting tools |
| `patterns.md` | Protocol classes, dataclasses, context managers |
| `security.md` | Secrets via env vars, bandit scanning |
| `testing.md` | pytest, 80%+ coverage, test markers |
| `hooks.md` | Auto-format after edit, mypy, print() warnings |

**Rules vs CLAUDE.md**: CLAUDE.md = always on for everything. Rules = activate conditionally by file type.

---

### Skills — Deep Knowledge On Demand

**Location**: `skills/` → copy to `.claude/skills/`

**Big difference from rules**: skills are deep reference documents (hundreds of lines) that Claude pulls in when the topic matches. They contain patterns, code examples, configuration templates, anti-patterns.

You don't call skills manually. Claude reads their descriptions and pulls them when relevant:

```
You: "Create a FastAPI endpoint for user registration"

Claude thinks:
  → fastapi-patterns (routing, DI, error handling)
  → pydantic-patterns (request/response models)
  → sqlalchemy-patterns (database models, async sessions)
```

**20 skills total:**

| Category | Skills |
|----------|--------|
| Core Python | `python-patterns`, `python-testing`, `pydantic-patterns`, `async-http-patterns` |
| Frameworks | `fastapi-patterns`, `django-patterns`, `django-security`, `django-tdd`, `django-verification`, `celery-patterns` |
| Data & Infra | `sqlalchemy-patterns`, `postgres-patterns`, `clickhouse-io`, `redis-patterns`, `database-migrations`, `deployment-patterns`, `docker-patterns` |
| QA Automation | `pytest-oop-patterns`, `allure-reporting`, `api-testing-patterns` |

**Example — what's inside `fastapi-patterns`**:
- Project structure (app factory, routers, services, repos)
- Pydantic Settings for config
- Dependency injection patterns (`Depends()`)
- Request/Response schemas with validation
- Error handling (custom exceptions, handlers)
- Middleware (logging, CORS)
- Background tasks, WebSockets
- JWT authentication
- Testing with httpx AsyncClient

---

### Agents — Specialized Sub-Brains

**Location**: `agents/` → copy to `.claude/agents/`

Agents are **specialized personas** Claude can spawn. Each has specific tools, a model (opus/sonnet/haiku), and a focused mission.

**11 agents:**

| Agent | Model | What It Does |
|-------|-------|-------------|
| `architect` | opus | Designs system architecture, evaluates trade-offs |
| `planner` | opus | Creates phased implementation plans |
| `tdd-guide` | sonnet | Enforces test-first development (RED→GREEN→REFACTOR) |
| `python-reviewer` | sonnet | Reviews Python code: PEP 8, types, security, patterns |
| `code-reviewer` | sonnet | Universal code review: security, quality, structure |
| `security-reviewer` | sonnet | OWASP Top 10, secrets, dependency CVEs |
| `database-reviewer` | sonnet | PostgreSQL: queries, indexes, schema, N+1 |
| `refactor-cleaner` | sonnet | Dead code detection and safe removal |
| `doc-updater` | haiku | Documentation generation from code |
| `qa-architect` | opus | Test framework design, OOP architecture |
| `api-test-writer` | sonnet | Generates OOP API tests with httpx + Allure |
| `qa-reviewer` | sonnet | Reviews test code: OOP compliance, Allure, flakiness |

**Model selection**: opus = deep thinking (slower, smarter), sonnet = fast execution, haiku = lightweight tasks.

---

### Commands — Your Slash-Command Toolkit

**Location**: `commands/` → copy to `.claude/commands/`

Type `/command` in Claude Code to trigger an action. Each command typically invokes an agent.

**11 commands:**

```
Planning & Development:
  /plan            Create implementation plan, WAIT for confirmation
  /tdd             Test-first: write failing test → implement → refactor
  /build-fix       Fix mypy/ruff errors one by one, minimal diffs
  /docs            Look up library docs ("FastAPI middleware")

Quality & Review:
  /code-review     Universal security & quality review
  /python-review   Python-specific: PEP 8, types, security
  /qa-review       Test code: OOP compliance, Allure, flaky patterns
  /verify          Full pipeline: types → lint → tests → security → git

Testing:
  /api-test        Generate OOP API test suite with Allure
  /test-coverage   Find gaps, generate tests to reach 80%+

Maintenance:
  /refactor-clean  Detect and safely remove dead code
```

**Example workflows:**

```bash
# New feature from scratch
/plan → confirm → /tdd → /python-review → /verify → push

# Write tests for existing API
/api-test POST /api/v1/orders → /qa-review → /verify

# Before PR
/verify pre-pr

# Clean up codebase
/refactor-clean → /verify
```

---

### Hooks — Automatic Actions

**Location**: `hooks/hooks.json` → merge into `.claude/hooks.json`

Hooks run automatically on events. You don't call them.

| When | What Happens |
|------|-------------|
| After editing `.py` | `ruff --fix` + `black` auto-format |
| After editing `.py` | `mypy` type check (async, non-blocking) |
| After editing `.py` | Warn if `print()` found → suggest `logging` |
| When Claude stops | Scan for `pdb`, `breakpoint()`, `print()` in changed files |

**Like a local CI pipeline running in real-time.**

---

### MCP Configs — External Integrations

**Location**: `mcp-configs/mcp-servers.json`

Give Claude Code access to external tools:

| Server | Purpose |
|--------|---------|
| Context7 | Live docs for any library (FastAPI, Django, SQLAlchemy) |
| GitHub | PRs, issues, repo operations |
| ClickHouse | Analytics queries |
| Playwright | Browser automation / E2E testing |
| InsAIts | AI security monitoring |

---

## How Everything Works Together

```
You type: /plan Add order processing with Stripe webhooks

    CLAUDE.md loads          → Claude knows all project rules
    planner agent spawns     → Creates phased implementation plan
    You confirm              → Claude starts coding

    fastapi-patterns skill   → Knows how to structure endpoints
    sqlalchemy-patterns      → Knows async DB patterns
    pydantic-patterns        → Knows how to build request/response models

    After each file edit:
      hooks run              → ruff + black format, mypy checks types

You type: /api-test POST /api/v1/orders

    api-test-writer agent    → Generates full test suite
    pytest-oop-patterns      → BaseTest, service classes, fixtures
    allure-reporting         → @epic, @feature, @story, @step
    api-testing-patterns     → HTTPClient wrapper, DataGenerator

You type: /verify pre-pr

    → mypy .                 → Types OK
    → ruff check .           → Lint OK
    → pytest --cov           → 94% coverage, 52/52 passed
    → bandit -r src/         → 0 vulnerabilities
    → Ready for PR: YES
```

---

## Installation

### Option 1: Copy what you need

```bash
cp CLAUDE.md your-project/
cp -r rules/ your-project/.claude/rules/python/
cp -r agents/ your-project/.claude/agents/
cp -r commands/ your-project/.claude/commands/
cp -r skills/fastapi-patterns/ your-project/.claude/skills/
cp -r skills/pytest-oop-patterns/ your-project/.claude/skills/
# ... pick your skills
```

### Option 2: Copy everything

```bash
cp CLAUDE.md your-project/
cp -r rules/ your-project/.claude/rules/python/
cp -r agents/ your-project/.claude/agents/
cp -r commands/ your-project/.claude/commands/
cp -r skills/ your-project/.claude/skills/
cp hooks/hooks.json your-project/.claude/hooks.json
```

### Option 3: Symlink (shared across projects)

```bash
ln -s /path/to/python-stack/agents/ your-project/.claude/agents
ln -s /path/to/python-stack/skills/ your-project/.claude/skills
ln -s /path/to/python-stack/commands/ your-project/.claude/commands
```

---

## Tech Stack Coverage

```
Python 3.9+
├── FastAPI          (async web framework)
├── Django + DRF     (full-stack web framework)
├── SQLAlchemy 2.0   (async ORM + Alembic migrations)
├── Pydantic v2      (data validation & settings)
├── Celery           (distributed task queues)
├── pytest           (testing framework)
├── httpx / aiohttp  (HTTP clients)
├── PostgreSQL       (primary database)
├── ClickHouse       (analytics database)
├── Redis            (caching, queues, locks)
├── Docker           (containerization)
├── Allure           (test reporting)
└── Tools: black, ruff, mypy, bandit, isort, faker
```

---

## Quick Reference

```
CLAUDE.md           = Project rules (always loaded, the "brain")
.claude/rules/      = Auto-activate by file type (edit .py → Python rules)
.claude/skills/     = Deep knowledge (Claude pulls when topic matches)
.claude/agents/     = Specialized sub-agents (invoked by commands)
.claude/commands/   = Slash commands (/plan, /tdd, /verify, /api-test)
.claude/hooks.json  = Auto-actions (format on save, type check)
mcp-configs/        = External integrations (GitHub, docs, ClickHouse)
```

---
---

<a id="russian"></a>

# Python Stack for Claude Code — Русская версия

## Что это?

**Готовый набор инструментов**, который превращает Claude Code в senior Python-инженера. Когда вы подключаете этот набор к проекту, Claude Code мгновенно знает как:

- Писать код по PEP 8 с type hints и правильной обработкой ошибок
- Проектировать архитектуры на FastAPI/Django/SQLAlchemy
- Генерировать ООП-тесты с pytest, httpx и Allure-отчётами
- Ревьюить код на уязвимости (OWASP Top 10)
- Работать с PostgreSQL, Redis, ClickHouse, Celery, Docker

**Без этого набора** Claude Code — умный генералист. **С ним** — senior Python-инженер, который следует стандартам вашей команды.

---

## Как работает конфигурация Claude Code

Когда вы запускаете Claude Code в директории проекта, он читает специальные файлы из папки `.claude/`:

```
your-project/
├── CLAUDE.md              # Главный файл инструкций — "мозг"
├── .claude/
│   ├── rules/             # Правила (активируются автоматически по типу файла)
│   ├── skills/            # Глубокие знания (подключаются по контексту)
│   ├── agents/            # Специализированные суб-агенты
│   ├── commands/          # Слэш-команды (/plan, /tdd, /verify)
│   └── hooks.json         # Автоматические действия (форматирование, проверки)
└── src/
```

Этот тулкит предоставляет все эти компоненты, настроенные для Python.

---

## Компоненты

### CLAUDE.md — Мозг

**Самый важный файл.** Лежит в корне проекта. Claude читает его в начале каждого разговора и следует как закону.

Наш CLAUDE.md содержит:
- Конвенции Python (PEP 8, type hints, black/ruff/mypy)
- Строгие правила ООП-тестов (классы, сервисные слои, Allure — обязательно)
- Правила по фреймворкам (FastAPI, Django, SQLAlchemy, Celery)
- Правила по БД (PostgreSQL, индексы, предотвращение N+1)
- Правила безопасности (никакого eval, никаких хардкодов секретов)
- Все доступные команды и агенты

**Представьте**: это как документ для онбординга нового разработчика. Всё, что он должен всегда соблюдать.

---

### Rules — Правила (Всегда активны)

**Расположение**: `rules/` → копировать в `.claude/rules/`

Markdown-файлы, которые **автоматически активируются** по паттерну файлов. Редактируете `.py` файл — включаются Python-правила.

```yaml
---
paths:
  - "**/*.py"
---
# Python Coding Style
- Следовать PEP 8
- Type annotations на всех публичных функциях
```

| Файл | Что контролирует |
|------|-----------------|
| `coding-style.md` | PEP 8, type hints, иммутабельность, black/ruff/isort |
| `patterns.md` | Protocol, dataclasses, контекстные менеджеры |
| `security.md` | Секреты через env vars, bandit-сканирование |
| `testing.md` | pytest, покрытие 80%+, маркеры тестов |
| `hooks.md` | Автоформат после редактирования, mypy, предупреждения о print() |

**Разница с CLAUDE.md**: CLAUDE.md = всегда включён для всего. Rules = включаются условно по типу файла.

---

### Skills — Глубокие знания по запросу

**Расположение**: `skills/` → копировать в `.claude/skills/`

**Главное отличие от правил**: скиллы — это большие справочные документы (сотни строк) с паттернами, примерами кода, конфигурациями, антипаттернами. Claude подтягивает их когда тема совпадает.

Вы не вызываете скиллы вручную. Claude сам решает когда они нужны:

```
Вы: "Создай FastAPI эндпоинт для регистрации пользователя"

Claude думает:
  → fastapi-patterns (роутинг, DI, обработка ошибок)
  → pydantic-patterns (модели запроса/ответа)
  → sqlalchemy-patterns (модели БД, async сессии)
```

**20 скиллов:**

| Категория | Скиллы |
|-----------|--------|
| Core Python | `python-patterns`, `python-testing`, `pydantic-patterns`, `async-http-patterns` |
| Фреймворки | `fastapi-patterns`, `django-patterns`, `django-security`, `django-tdd`, `django-verification`, `celery-patterns` |
| Данные и инфра | `sqlalchemy-patterns`, `postgres-patterns`, `clickhouse-io`, `redis-patterns`, `database-migrations`, `deployment-patterns`, `docker-patterns` |
| QA автоматизация | `pytest-oop-patterns`, `allure-reporting`, `api-testing-patterns` |

**Пример — что внутри `fastapi-patterns`:**
- Структура проекта (app factory, routers, services, repos)
- Pydantic Settings для конфигурации
- Паттерны Dependency Injection (`Depends()`)
- Схемы Request/Response с валидацией
- Обработка ошибок (кастомные исключения)
- Middleware (логирование, CORS)
- Background tasks, WebSockets
- JWT аутентификация
- Тестирование через httpx AsyncClient

---

### Agents — Специализированные суб-агенты

**Расположение**: `agents/` → копировать в `.claude/agents/`

Агенты — это **специализированные персоны**, которых Claude может вызвать. У каждого свои инструменты, своя модель и своя миссия.

**11 агентов:**

| Агент | Модель | Что делает |
|-------|--------|-----------|
| `architect` | opus | Проектирует архитектуру, оценивает trade-offs |
| `planner` | opus | Создаёт пошаговые планы реализации |
| `tdd-guide` | sonnet | Контролирует TDD: RED → GREEN → REFACTOR |
| `python-reviewer` | sonnet | Ревью Python: PEP 8, типы, безопасность |
| `code-reviewer` | sonnet | Универсальный ревью: безопасность, качество |
| `security-reviewer` | sonnet | OWASP Top 10, секреты, CVE в зависимостях |
| `database-reviewer` | sonnet | PostgreSQL: запросы, индексы, схема, N+1 |
| `refactor-cleaner` | sonnet | Поиск и безопасное удаление мёртвого кода |
| `doc-updater` | haiku | Генерация документации из кода |
| `qa-architect` | opus | Проектирование тест-фреймворка, ООП архитектура |
| `api-test-writer` | sonnet | Генерация ООП API-тестов с httpx + Allure |
| `qa-reviewer` | sonnet | Ревью тест-кода: ООП, Allure, flaky-паттерны |

**Выбор модели**: opus = глубокое мышление (медленнее, умнее), sonnet = быстрое выполнение, haiku = лёгкие задачи.

---

### Commands — Слэш-команды

**Расположение**: `commands/` → копировать в `.claude/commands/`

Набираете `/команда` в Claude Code — запускается действие. Каждая команда обычно вызывает агента.

**11 команд:**

```
Планирование и разработка:
  /plan            Создать план реализации, ЖДАТЬ подтверждения
  /tdd             Тест-первый подход: RED → GREEN → REFACTOR
  /build-fix       Починить ошибки mypy/ruff по одной
  /docs            Посмотреть актуальную документацию библиотеки

Качество и ревью:
  /code-review     Универсальный ревью безопасности и качества
  /python-review   Python-специфичный: PEP 8, типы, безопасность
  /qa-review       Тест-код: ООП, Allure, flaky-паттерны
  /verify          Полный пайплайн: типы → линт → тесты → безопасность

Тестирование:
  /api-test        Сгенерировать ООП тест-сьюит с Allure
  /test-coverage   Найти пробелы, дописать тесты до 80%+

Обслуживание:
  /refactor-clean  Найти и безопасно удалить мёртвый код
```

**Примеры рабочих процессов:**

```bash
# Новая фича с нуля
/plan → подтверждаем → /tdd → /python-review → /verify → push

# Написать тесты для существующего API
/api-test POST /api/v1/orders → /qa-review → /verify

# Перед PR
/verify pre-pr

# Навести порядок в коде
/refactor-clean → /verify
```

---

### Hooks — Автоматические действия

**Расположение**: `hooks/hooks.json` → добавить в `.claude/hooks.json`

Хуки срабатывают автоматически на события. Вы их не вызываете.

| Когда | Что происходит |
|-------|---------------|
| После редактирования `.py` | `ruff --fix` + `black` автоформат |
| После редактирования `.py` | `mypy` проверка типов (в фоне) |
| После редактирования `.py` | Предупреждение если найден `print()` → предложит `logging` |
| Когда Claude останавливается | Поиск `pdb`, `breakpoint()`, `print()` в изменённых файлах |

**Как локальный CI-пайплайн, работающий в реальном времени.**

---

### MCP Configs — Внешние интеграции

**Расположение**: `mcp-configs/mcp-servers.json`

Дают Claude Code доступ к внешним инструментам:

| Сервер | Для чего |
|--------|----------|
| Context7 | Актуальные доки любой библиотеки (FastAPI, Django, SQLAlchemy) |
| GitHub | PR, issues, операции с репозиториями |
| ClickHouse | Аналитические запросы |
| Playwright | Автоматизация браузера / E2E тесты |
| InsAIts | AI-мониторинг безопасности |

---

## Как всё работает вместе

```
Вы: /plan Добавить обработку заказов с Stripe webhooks

    CLAUDE.md загружается     → Claude знает все правила проекта
    planner агент запускается  → Создаёт план по фазам
    Вы подтверждаете           → Claude начинает кодить

    fastapi-patterns скилл    → Знает как структурировать эндпоинты
    sqlalchemy-patterns        → Знает async паттерны БД
    pydantic-patterns          → Знает как строить модели

    После каждого сохранения:
      хуки работают            → ruff + black форматируют, mypy проверяет типы

Вы: /api-test POST /api/v1/orders

    api-test-writer агент      → Генерирует полный тест-сьюит
    pytest-oop-patterns        → BaseTest, сервисные классы, fixtures
    allure-reporting           → @epic, @feature, @story, @step
    api-testing-patterns       → HTTPClient wrapper, DataGenerator

Вы: /verify pre-pr

    → mypy .                   → Типы OK
    → ruff check .             → Линт OK
    → pytest --cov             → 94% покрытие, 52/52 прошли
    → bandit -r src/           → 0 уязвимостей
    → Готов к PR: ДА
```

---

## Установка

### Вариант 1: Скопировать нужное

```bash
cp CLAUDE.md your-project/
cp -r rules/ your-project/.claude/rules/python/
cp -r agents/ your-project/.claude/agents/
cp -r commands/ your-project/.claude/commands/
cp -r skills/fastapi-patterns/ your-project/.claude/skills/
cp -r skills/pytest-oop-patterns/ your-project/.claude/skills/
# ... выберите нужные скиллы
```

### Вариант 2: Скопировать всё

```bash
cp CLAUDE.md your-project/
cp -r rules/ your-project/.claude/rules/python/
cp -r agents/ your-project/.claude/agents/
cp -r commands/ your-project/.claude/commands/
cp -r skills/ your-project/.claude/skills/
cp hooks/hooks.json your-project/.claude/hooks.json
```

### Вариант 3: Симлинки (общие для нескольких проектов)

```bash
ln -s /path/to/python-stack/agents/ your-project/.claude/agents
ln -s /path/to/python-stack/skills/ your-project/.claude/skills
ln -s /path/to/python-stack/commands/ your-project/.claude/commands
```

---

## Покрытие технологий

```
Python 3.9+
├── FastAPI          (async веб-фреймворк)
├── Django + DRF     (full-stack веб-фреймворк)
├── SQLAlchemy 2.0   (async ORM + Alembic миграции)
├── Pydantic v2      (валидация данных и настройки)
├── Celery           (распределённые очереди задач)
├── pytest           (тестирование)
├── httpx / aiohttp  (HTTP-клиенты)
├── PostgreSQL       (основная БД)
├── ClickHouse       (аналитическая БД)
├── Redis            (кэш, очереди, блокировки)
├── Docker           (контейнеризация)
├── Allure           (отчёты по тестам)
└── Инструменты: black, ruff, mypy, bandit, isort, faker
```

---

## Шпаргалка

```
CLAUDE.md           = Правила проекта (всегда загружен, "мозг")
.claude/rules/      = Авто-активация по типу файла (редактируешь .py → Python правила)
.claude/skills/     = Глубокие знания (Claude подтягивает когда тема совпадает)
.claude/agents/     = Специализированные суб-агенты (вызываются командами)
.claude/commands/   = Слэш-команды (/plan, /tdd, /verify, /api-test)
.claude/hooks.json  = Авто-действия (форматирование, проверка типов)
mcp-configs/        = Внешние интеграции (GitHub, документация, ClickHouse)
```

---

## Directory Structure

```
python-stack/
├── README.md                           # This file (EN + RU tutorial)
├── CLAUDE.md                           # Main brain — copy to project root
│
├── rules/                              # 5 auto-activate rules for .py files
│   ├── coding-style.md
│   ├── hooks.md
│   ├── patterns.md
│   ├── security.md
│   └── testing.md
│
├── skills/                             # 20 deep knowledge skills
│   ├── python-patterns/SKILL.md
│   ├── python-testing/SKILL.md
│   ├── pydantic-patterns/SKILL.md
│   ├── fastapi-patterns/SKILL.md
│   ├── sqlalchemy-patterns/SKILL.md
│   ├── celery-patterns/SKILL.md
│   ├── redis-patterns/SKILL.md
│   ├── async-http-patterns/SKILL.md
│   ├── django-patterns/SKILL.md
│   ├── django-security/SKILL.md
│   ├── django-tdd/SKILL.md
│   ├── django-verification/SKILL.md
│   ├── postgres-patterns/SKILL.md
│   ├── clickhouse-io/SKILL.md
│   ├── docker-patterns/SKILL.md
│   ├── database-migrations/SKILL.md
│   ├── deployment-patterns/SKILL.md
│   ├── pytest-oop-patterns/SKILL.md
│   ├── allure-reporting/SKILL.md
│   └── api-testing-patterns/SKILL.md
│
├── agents/                             # 11 specialized agents
│   ├── architect.md
│   ├── planner.md
│   ├── tdd-guide.md
│   ├── python-reviewer.md
│   ├── code-reviewer.md
│   ├── security-reviewer.md
│   ├── database-reviewer.md
│   ├── refactor-cleaner.md
│   ├── doc-updater.md
│   ├── qa-architect.md
│   ├── api-test-writer.md
│   └── qa-reviewer.md
│
├── commands/                           # 11 slash commands
│   ├── plan.md
│   ├── tdd.md
│   ├── build-fix.md
│   ├── docs.md
│   ├── code-review.md
│   ├── python-review.md
│   ├── qa-review.md
│   ├── verify.md
│   ├── api-test.md
│   ├── test-coverage.md
│   └── refactor-clean.md
│
├── hooks/
│   └── hooks.json                      # Auto-format, mypy, print warnings
│
├── mcp-configs/
│   └── mcp-servers.json                # GitHub, Context7, ClickHouse, Playwright
│
└── docs/
    └── TUTORIAL_EN.md                  # Extended English tutorial
```
