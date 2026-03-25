# Python Stack for Claude Code — Complete Tutorial

## What Is This?

This is a **ready-to-use toolkit** that supercharges Claude Code for Python development and QA automation. Think of it as a "brain transplant" — once you drop it into your project, Claude Code instantly knows how to:

- Write Pythonic code following PEP 8, with type hints, proper error handling
- Design FastAPI/Django/SQLAlchemy architectures from scratch
- Generate OOP test suites with pytest, httpx, Allure reports
- Review code for security vulnerabilities (OWASP Top 10)
- Manage PostgreSQL, Redis, ClickHouse, Celery, Docker
- And dozens of other things, without you explaining anything

**Without this toolkit**, Claude Code is a smart generalist. **With it**, Claude Code becomes a senior Python engineer who follows your team's exact standards.

---

## How Claude Code Works (The Basics)

When you run Claude Code in a project directory, it reads special files to understand context:

```
your-project/
├── CLAUDE.md              <-- Main instructions file (like a "brain" for Claude)
├── .claude/               <-- Claude Code configuration directory
│   ├── rules/             <-- Always-active guidelines
│   ├── skills/            <-- Knowledge activated on demand
│   ├── agents/            <-- Specialized sub-agents (reviewers, planners, etc.)
│   ├── commands/          <-- Slash commands you type (/plan, /tdd, /verify)
│   └── hooks.json         <-- Automations that run on events
└── src/                   <-- Your actual code
```

Claude reads these files and adjusts its behavior accordingly. Let's break down each component.

---

## CLAUDE.md — The Brain

`CLAUDE.md` is the **single most important file**. It sits in your project root and Claude Code reads it at the start of every conversation.

### What goes in CLAUDE.md?

Everything Claude must ALWAYS follow:

```markdown
# CLAUDE.md

## Mandatory Conventions
- ALL tests MUST be in classes inheriting BaseTest
- ALL API calls through service classes, never raw httpx
- Use logging, NEVER print()
- Type annotations on all public functions

## Running Tests
pytest --alluredir=allure-results -v

## Key Commands
| Command | Purpose |
|---------|---------|
| /plan   | Plan before coding |
| /tdd    | Write tests first |
```

**Key insight**: CLAUDE.md is like an onboarding document for a new developer. Write it like you're explaining your project rules to someone joining the team.

### Our CLAUDE.md includes:
- Python code conventions (PEP 8, typing, formatting)
- Strict OOP test rules (classes, service layers, Allure)
- Framework-specific rules (FastAPI, Django, SQLAlchemy)
- Database rules (PostgreSQL, indexing, N+1 prevention)
- Security rules (no eval, no hardcoded secrets)
- All available commands and agents

---

## Rules — Always-On Guidelines

**Location**: `.claude/rules/` (or `rules/` in our toolkit)

Rules are markdown files that **automatically activate** based on file patterns. When you edit a `.py` file, Python rules kick in. When you edit `.sql`, database rules activate.

### How rules work

Each rule file has a frontmatter specifying when it activates:

```yaml
---
paths:
  - "**/*.py"
  - "**/*.pyi"
---
# Python Coding Style

- Follow PEP 8
- Use type annotations on all function signatures
- Use black for formatting, ruff for linting
```

### Our Python rules (5 files):

| Rule | What it enforces |
|------|-----------------|
| `coding-style.md` | PEP 8, type hints, black/ruff/isort, immutability |
| `patterns.md` | Protocol classes, dataclasses, context managers, generators |
| `security.md` | Secret management via env vars, bandit scanning |
| `testing.md` | pytest as framework, 80%+ coverage, test markers |
| `hooks.md` | Auto-format after edit, mypy checking, print() warnings |

### When to use rules vs CLAUDE.md?

- **CLAUDE.md** = project-wide rules that apply to everything
- **Rules** = language/context-specific rules that activate conditionally

---

## Skills — On-Demand Knowledge

**Location**: `.claude/skills/` (or `skills/` in our toolkit)

Skills are **deep knowledge files** on specific topics. Unlike rules (always on), skills activate when the context matches — Claude pulls them in when it needs that knowledge.

### Skill structure

Each skill lives in its own directory with a `SKILL.md` file:

```
skills/
├── fastapi-patterns/
│   └── SKILL.md          <-- FastAPI knowledge (800+ lines of patterns)
├── sqlalchemy-patterns/
│   └── SKILL.md          <-- SQLAlchemy 2.0 patterns
└── celery-patterns/
    └── SKILL.md          <-- Celery task queue patterns
```

### What's inside a skill?

```yaml
---
name: fastapi-patterns
description: FastAPI patterns for building production-grade APIs
---

# FastAPI Development Patterns

## When to Activate
- Building FastAPI web applications
- Designing REST APIs with FastAPI

## Project Structure
(detailed directory layout)

## App Factory
(code example with create_app())

## Dependency Injection
(Depends() patterns, database sessions, auth)

## Router Patterns
(CRUD endpoints, pagination, filtering)

...hundreds of lines of battle-tested patterns
```

### Our skills (20 total):

**Core Python** (4): python-patterns, python-testing, pydantic-patterns, async-http-patterns

**Frameworks** (6): fastapi-patterns, django-patterns, django-security, django-tdd, django-verification, celery-patterns

**Data & Infrastructure** (7): sqlalchemy-patterns, postgres-patterns, clickhouse-io, redis-patterns, database-migrations, deployment-patterns, docker-patterns

**QA Automation** (3): pytest-oop-patterns, allure-reporting, api-testing-patterns

### How skills get used

You don't need to "call" skills. Claude Code reads their descriptions and pulls them when relevant:

```
You: "Create a FastAPI endpoint for user registration"

Claude thinks: "This involves FastAPI → let me use fastapi-patterns skill"
              "Need Pydantic models → pydantic-patterns skill"
              "Need database → sqlalchemy-patterns skill"
```

The skill gives Claude hundreds of lines of battle-tested patterns to follow, instead of guessing.

---

## Agents — Specialized Sub-Brains

**Location**: `.claude/agents/` (or `agents/` in our toolkit)

Agents are **specialized personas** that Claude can invoke. Each agent has specific tools, a specific model, and a specific mission.

### Agent structure

```yaml
---
name: python-reviewer
description: Expert Python code reviewer specializing in PEP 8, type hints, security
tools: ["Read", "Grep", "Glob", "Bash"]
model: sonnet
---

You are a senior Python code reviewer...

## Review Priorities
### CRITICAL — Security
- SQL Injection: f-strings in queries
- Command Injection: unvalidated input in shell commands
...
```

### Our agents (11 total):

**Architecture & Planning:**
| Agent | Model | Mission |
|-------|-------|---------|
| `architect` | opus | System design, scalability, trade-offs, ADRs |
| `planner` | opus | Step-by-step implementation plans with risk assessment |

**Code Quality:**
| Agent | Model | Mission |
|-------|-------|---------|
| `python-reviewer` | sonnet | PEP 8, type hints, Pythonic patterns, security |
| `code-reviewer` | sonnet | Universal quality: security, structure, best practices |
| `security-reviewer` | sonnet | OWASP Top 10, secrets, dependency vulnerabilities |
| `database-reviewer` | sonnet | PostgreSQL queries, indexes, schema, N+1 detection |
| `refactor-cleaner` | sonnet | Dead code detection, safe removal with test verification |
| `doc-updater` | haiku | Auto-generate documentation from code |

**QA Automation:**
| Agent | Model | Mission |
|-------|-------|---------|
| `qa-architect` | opus | Test framework design, OOP architecture, pytest infra |
| `api-test-writer` | sonnet | Generate OOP API tests with httpx, Allure, factories |
| `qa-reviewer` | sonnet | Review tests: OOP compliance, Allure, flaky patterns |

### How agents get used

Agents are typically invoked through commands (next section), or Claude may use them proactively:

```
You: "/python-review"
Claude: → spawns python-reviewer agent
        → agent runs git diff, ruff, mypy, bandit
        → returns categorized report (CRITICAL/HIGH/MEDIUM)
```

### Model selection matters

- **opus** = deep thinking (architecture, planning) — slower but smarter
- **sonnet** = fast execution (reviews, code generation) — quick and accurate
- **haiku** = lightweight tasks (documentation) — fastest and cheapest

---

## Commands — Your Slash-Command Toolkit

**Location**: `.claude/commands/` (or `commands/` in our toolkit)

Commands are **actions you trigger** by typing `/command-name` in Claude Code. Each command is a markdown file describing what to do.

### Command structure

```yaml
---
description: Review test code for OOP compliance and Allure metadata
---

# QA Test Code Review

This command invokes the **qa-reviewer** agent...

## What This Command Does
1. Check OOP compliance — all tests in classes
2. Check Allure metadata — @epic, @feature, @story
3. Detect flaky patterns — time.sleep(), shared state
...
```

### Our commands (11 total):

**Planning & Development:**
```
/plan          → Create implementation plan, wait for confirmation
/tdd           → Test-driven development: RED → GREEN → REFACTOR
/build-fix     → Fix mypy/ruff errors one by one with minimal diffs
/docs          → Look up current library docs via Context7
```

**Quality & Review:**
```
/code-review     → Universal security & quality review
/python-review   → Python-specific review (PEP 8, types, security)
/qa-review       → Test code review (OOP, Allure, flaky patterns)
/verify          → Full pipeline: types + lint + tests + security
```

**Testing:**
```
/api-test        → Generate OOP API test suite with Allure
/test-coverage   → Find gaps, generate tests to reach 80%+
```

**Maintenance:**
```
/refactor-clean  → Detect and safely remove dead code
```

### Usage examples

```bash
# Planning a feature
You: /plan Add WebSocket notifications for order status updates

# Writing tests
You: /api-test POST /api/v1/orders -- needs auth, creates order from cart

# Reviewing before PR
You: /verify pre-pr

# Checking test quality
You: /qa-review

# Looking up docs
You: /docs "SQLAlchemy" "async session setup"
```

---

## Hooks — Automatic Actions

**Location**: `.claude/hooks.json` (or `hooks/hooks.json` in our toolkit)

Hooks are **automations that trigger on events** — like "after editing a file" or "when Claude stops responding". You don't call them; they run automatically.

### Hook structure

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "...run ruff + black on edited .py file..."
          }
        ],
        "description": "Auto-format Python files after edits"
      }
    ]
  }
}
```

### Our hooks:

| Event | What happens |
|-------|-------------|
| After editing `.py` file | `ruff check --fix` + `black` auto-format |
| After editing `.py` file | `mypy` type check (async, non-blocking) |
| After editing `.py` file | Warn if `print()` found, suggest `logging` |
| When Claude stops | Check modified files for `pdb`, `breakpoint()`, `print()` |

### Why hooks matter

Without hooks, you'd have to manually ask Claude to format code or run mypy after every edit. Hooks make it automatic — like having a CI pipeline running locally in real-time.

---

## MCP Configs — External Integrations

**Location**: `mcp-configs/mcp-servers.json`

MCP (Model Context Protocol) servers give Claude Code access to external tools and services.

### Our configured MCPs:

| Server | What it does |
|--------|-------------|
| **Context7** | Look up current docs for any library (FastAPI, Django, etc.) |
| **GitHub** | Read/create PRs, issues, browse repos |
| **ClickHouse** | Run analytics queries directly |
| **Playwright** | Browser automation for E2E testing |
| **InsAIts** | AI security monitoring (Python-native) |

### How to enable

Copy the servers you need into your `~/.claude.json`:

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"]
    }
  }
}
```

---

## How Everything Works Together

Here's a typical workflow showing all pieces in action:

### Scenario: Build a new API endpoint

```
Step 1: /plan Add user registration endpoint with email verification

  → planner agent (opus) analyzes codebase
  → Creates phased plan: models → service → endpoint → tests
  → WAITS for your confirmation

Step 2: You confirm → /tdd

  → tdd-guide agent activates
  → Writes failing test first (RED)
  → fastapi-patterns skill provides endpoint patterns
  → pydantic-patterns skill provides model patterns
  → sqlalchemy-patterns skill provides DB patterns
  → Implements minimal code (GREEN)
  → Refactors (REFACTOR)

Step 3: After coding, hooks run automatically

  → ruff + black format the new .py files
  → mypy checks types in background
  → Warns about any print() statements

Step 4: /python-review

  → python-reviewer agent scans changes
  → Checks PEP 8, type hints, security, Pythonic patterns
  → Reports: "0 CRITICAL, 0 HIGH, 1 MEDIUM (missing docstring)"

Step 5: /qa-review

  → qa-reviewer agent checks test code
  → Verifies OOP compliance, Allure metadata, no flaky patterns
  → Reports: "All tests in classes, Allure complete, no sleep()"

Step 6: /verify pre-pr

  → Runs: mypy → ruff → pytest → bandit → pip-audit → git status
  → Reports: "Types OK, Lint OK, 47/47 passed (92% coverage), 0 vulns"
  → "Ready for PR: YES"
```

### Scenario: Write API tests for existing endpoints

```
Step 1: /api-test GET /api/v1/products -- paginated, filterable by category

  → api-test-writer agent activates
  → Creates: Pydantic models, ProductsAPI service class, TestProductsList class
  → Covers: positive (list, filter, paginate), negative (bad params, 401), edge cases
  → All with @allure.epic, @feature, @story, @severity

Step 2: /qa-review

  → Verifies all tests are OOP-compliant
  → Checks Allure is complete
  → No hardcoded data, no sleep(), no raw httpx calls

Step 3: Run tests
  pytest --alluredir=allure-results -v
  allure serve allure-results
  → Beautiful HTML report with steps, attachments, request/response bodies
```

---

## Installation Guide

### Option 1: Copy what you need

```bash
# Copy to your Python project
cp CLAUDE.md your-project/
cp -r rules/ your-project/.claude/rules/python/
cp -r agents/ your-project/.claude/agents/
cp -r commands/ your-project/.claude/commands/
cp -r hooks/ your-project/.claude/

# Copy only the skills you need
cp -r skills/fastapi-patterns/ your-project/.claude/skills/
cp -r skills/pytest-oop-patterns/ your-project/.claude/skills/
cp -r skills/allure-reporting/ your-project/.claude/skills/
```

### Option 2: Copy everything

```bash
cp -r python-stack/ your-project/.claude-python-stack/
# Then reference from your CLAUDE.md
```

### Option 3: Symlink (for multiple projects)

```bash
ln -s /path/to/python-stack/agents/ your-project/.claude/agents
ln -s /path/to/python-stack/skills/ your-project/.claude/skills
ln -s /path/to/python-stack/commands/ your-project/.claude/commands
```

---

## Quick Reference Card

```
CLAUDE.md          = Project rules (always loaded)
.claude/rules/     = Language-specific rules (auto-activate by file type)
.claude/skills/    = Deep knowledge (activated when context matches)
.claude/agents/    = Specialized sub-agents (invoked by commands)
.claude/commands/  = Slash commands (/plan, /tdd, /verify)
.claude/hooks.json = Automatic actions (format on save, type check)
mcp-configs/       = External tool integrations (GitHub, docs, etc.)
```

**Remember**: You don't need to use everything. Start with `CLAUDE.md` + a few agents + the skills you need. Add more as your project grows.
