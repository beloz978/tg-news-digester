---
title: AI Dev Tools Zoomcamp final project — tg-news-digester
slug: ai-dev-tools-final-project
date: 2026-10-27
status: draft
jira: ""
author: Nikolay Krupiy
---

# AI Dev Tools Zoomcamp final project — tg-news-digester

## Context

Spec: https://github.com/DataTalksClub/ai-dev-tools-zoomcamp/tree/main/project (only `README.md`).
One app, built end to end with an agent workflow; ~20 h; 24 points, **10 to pass**; 2 attempts;
**3 peer reviews are mandatory** (no reviews = fail). Docs in English. Deadlines: course platform
(courses.datatalks.club/ai-dev-tools-2026/) — *not checked, fill in below*.

App idea (repo name): **Telegram news digester** — pulls items from a configured list of **info-sources**
grouped by subject (see "Info-sources feature"), has an LLM summarise them into a digest, stores digests in
Postgres, shows them in a web UI.
The key user journey for telemetry: *"generate today's digest for a subject"* (fetch sources → summarise → store → render).

## Decisions (change here if you disagree)

| Area | Choice | Why |
|---|---|---|
| Backend | Python 3.12 + FastAPI, `uv` | One-command run, easy OTel |
| Frontend | Server-rendered Jinja + HTMX | Smallest thing that is still a real frontend |
| Storage | Postgres | Persistent, in compose |
| Ingest | Pluggable fetchers by source `type`: `telegram` (`t.me/s/<channel>`, no login), `web` (HTML page), `rss` | Sources are not only Telegram (see below); no MTProto session to manage |
| LLM | Claude API, model id from config | Summaries |
| Observability | OpenTelemetry → `grafana/otel-lgtm` (one container: Grafana+Tempo+Loki+Prometheus) | Traces/logs/metrics + UI, shared trace ID |
| Deploy | Render (`render.yaml`, Docker) | Public URL, rollback by redeploying previous image |
| CI/CD | GitHub Actions | PR tests; auto-deploy staging; **manual** prod |
| On-call agent | Alert webhook → headless Claude Code/Agent SDK → opens fix PR | "Wakes on alert and fixes" |

Repo conventions that apply: work on `feat/<topic>` branches, small Conventional Commits, push/PR only after
user confirmation; no hardcoded IDs/URLs/timeouts/channel names — `config/config.yaml` + `config/.env.common`
+ `config/.env.secrets` (with `.env.secrets.demo` / `.env.template`), read strictly, **no defaults**.

## Rubric → deliverable map (target 24/24, floor 10)

| Criterion (pts) | Deliverable / evidence |
|---|---|
| Problem Description (2) | README: problem, features, expected behaviour, screenshot |
| Product Specification (2) | `docs/spec.md` + **public** GitHub Issues/Project with acceptance criteria; PRs reference issues (evidence agents used it) |
| Agent Team Specification (2) | `docs/agents/team.md`: roles (spec, implementer, tester, reviewer, on-call), hand-offs, who owns what |
| Context Management (2) | `AGENTS.md` (short) linking modular docs: `docs/architecture.md`, `docs/testing.md`, `docs/runbook.md`, `docs/agents/*` |
| Application (2) | FE+BE+DB; `just dev` (or `make dev`) runs locally |
| Testing (2) | pytest unit + integration (real Postgres via compose/testcontainers, LLM and Telegram faked); `just test` |
| Containerization (2) | `docker compose up` starts app + Postgres + otel-lgtm |
| Deployment (2) | Public Render URL linked in README (fallback: recorded demo) |
| CI/CD (3) | PR: lint+tests. Merge to main: build image, auto-deploy staging. Prod: `workflow_dispatch` behind a protected Environment. Documented rollback |
| Observability (2) | Journey "generate digest" emits spans, metrics, structured JSON logs carrying `trace_id`; redact post text/tokens; Grafana dashboard JSON committed |
| Alerting on user impact (1) | Grafana alert: digest-generation error rate or p95 latency over threshold |
| On-Call Agent (2, all-or-nothing) | Webhook service + agent prompt + recorded run: injected bug → alert → agent PR that fixes it |

## Фича: info-sources (каналы по теме `{subject}`)

Идея: дайджест строится **по теме** (`subject`), а у темы есть список источников. Источник — это не обязательно
Telegram-канал, поэтому у каждого есть `type`.

**Модель данных** (таблицы `subject`, `source`):
`subject(id, slug, title)` · `source(id, subject_id, type, url, title, enabled, last_fetched_at, last_error)`.
`type ∈ {telegram, web, rss}`; значения enum — в конфиге, не в коде.

**Конфиг** (`config/config.yaml`, секция `sources`, читается строго, без дефолтов; по правилу «скаляры не в коде»):

```yaml
sources:
  subjects:
    genai:
      title: "GenAI"
      items:
        - type: web
          url: "https://calendarity.now/"
          title: "Calendarity — календарь сингулярности"
```

При старте конфиг синхронизируется в БД (upsert по `url`); отсутствующая секция/ключ → ошибка с именем ключа и файлом.

**Поведение:**
- `fetch(source)` → список `Item(source_id, url, title, published_at, text)`; свой fetcher на каждый `type`, общий интерфейс.
- Падение одного источника **не валит** дайджест: пишем `last_error`, метрику `source_fetch_failures_total{type}`, идём дальше.
- Источник вернул 0 элементов — это нормальный исход (метрика `source_items_total=0`, в дайджесте пометка «нет новостей»), а не ошибка.
- UI: страница темы со списком источников и статусом (последний fetch, ошибка), кнопка «собрать дайджест».
- Телеметрия: span на каждый источник (`source.type`, `source.id`; **без текста постов**), это же даёт осмысленный сценарий для алерта и on-call агента.

**Первый источник — `https://calendarity.now/` (тема GenAI). ⚠ Проверка показала расхождение:** это экспериментальный
«календарь сингулярности» от Postnikov.ai (конвертер дат, эпоха 30.11.2022); раздел «События» **пуст** («Событий пока нет»),
RSS/Atom, Telegram-канала, sitemap и API не найдено, отдельных URL у новостей нет. Т. е. как новостной источник он сейчас
ничего не отдаст (возможно, события подгружаются JS-ом — fetch этого не покажет, нужно проверить в браузере).
Решение: оставляем его источником №1 по вашему запросу (fetcher `web`), но не опираемся на него как на единственный —
см. открытые вопросы. Это же удобный реальный тест на «0 элементов».

## Steps (≈20 h budget)

1. **Spec first (1.5 h)** — README problem statement, `docs/spec.md`, create public GitHub repo + issues with acceptance criteria, `AGENTS.md` skeleton, `docs/agents/team.md`.
2. **Skeleton + config (1.5 h)** — FastAPI app, config loader (strict, no defaults), Postgres schema/migrations, `.env.template`, `just dev`.
3. **Core journey (3.5 h)** — subjects/sources schema + config sync, fetchers (`web` first, then `telegram`/`rss`), summariser (behind an interface), digest storage, HTMX pages (subjects, sources status, list digests, trigger generation).
4. **Tests (2 h)** — unit for parsing/summary-prompt building; integration for the journey against Postgres; `just test`.
5. **Containerise (1 h)** — Dockerfile, `docker-compose.yml` (app, db, otel-lgtm), health endpoint.
6. **Observability (2.5 h)** — OTel SDK, spans around fetch/LLM/store, metrics (digests, failures, latency), JSON logs with trace ID, redaction, dashboard.
7. **Deploy (1.5 h)** — `render.yaml`, secrets in Render, public URL, smoke test.
8. **CI/CD (2.5 h)** — workflows: `ci.yml` (PR), `deploy-staging.yml` (push main), `deploy-prod.yml` (manual, environment approval); rollback runbook; verify with a throwaway PR.
9. **Alert (1 h)** — Grafana alert rule (provisioned as file), contact point = on-call webhook.
10. **On-call agent (3 h)** — webhook receiver, prompt + tool scope (read logs/traces, git, open PR; **no direct prod deploy**), seeded fault (e.g. summariser crashes on empty channel) to demo end to end; record evidence in `docs/oncall-demo.md`.
11. **Polish + submit (1 h)** — README pass against the rubric table above, submit, then do the **3 peer reviews**.

Order of risk: steps 10 and 8 are the expensive ones (10 is worth 2 all-or-nothing; 8 is worth 3). If time runs short, cut scope in 3 and 6, not in 8.

## Acceptance Criteria

- [ ] Every rubric row above has a linked file/URL in README "Evaluation checklist" section
- [ ] `just dev`, `just test`, `docker compose up` each work from a clean clone
- [ ] Public URL live; staging auto-deploys; prod deploy is manual
- [ ] Trace ID visible in logs and Tempo for one digest generation; no post text or secrets in telemetry
- [ ] Sources for subject `genai` come from `config/config.yaml`; adding a source needs no code change
- [ ] One failing source / one empty source does not break digest generation (covered by tests)
- [ ] Injected fault triggers alert → agent opens a fixing PR
- [ ] Submission made; 3 peer reviews done

## Open questions

- Submission/peer-review deadlines (read from the course platform).
- `calendarity.now` не содержит новостей (см. выше): добавить рядом реальный GenAI-источник (публичный Telegram-канал или RSS)?
  Иначе первый дайджест будет пустым. Нужен ваш выбор канала/ленты.
- Нужно ли LLM-«дедуплицировать» одну новость из нескольких источников (сейчас вне scope)?
- Is a Telegram *bot* needed (deliver digest to a chat), or is web-only enough? Plan assumes web-only; bot delivery is out of scope unless time remains.
- Render vs another host if the user already has credits elsewhere.

## References

- Spec: https://github.com/DataTalksClub/ai-dev-tools-zoomcamp/tree/main/project
- Course platform: https://courses.datatalks.club/ai-dev-tools-2026/
