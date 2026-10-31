# CLAUDE.md — werkafspraken voor Lead Pipeline

## Project
Zelf-gehoste CRM (leads, pipelines, Gmail-threadopvolging, optionele AI) als
Home Assistant add-on. Databank: SQLite in `/data` op de HA-server.
Ontwerp: `docs/ONTWERP.md` (leidend). Bouwplan: `CLAUDE_CODE_PROMPT.md`.
Beslissingen buiten het ontwerp: `docs/BESLISSINGEN.md`.

## Taal
- UI-teksten: Nederlands (België) via i18n, Engels als tweede taal.
- Code, identifiers, commits, code-commentaar: Engels.
- Documentatie voor de gebruiker (README, DOCS.md, docs/): Nederlands.

## Commando's (na Fase 0)
- `make dev` — volledige app lokaal (DEV_MODE, fake Gmail/HA/AI, seeddata)
- `make test` — backend (pytest) + frontend (vitest)
- `make lint` — ruff, mypy, eslint, prettier --check
- `make build` — add-on Docker-image (amd64)
- Migratie maken: `cd backend && uv run alembic revision --autogenerate -m "..."`

## Regels
- Nooit automatisch mails verzenden; AI wijzigt nooit zelf data.
- Nooit de volledige mailbox synchroniseren; enkel bekende contacten of label `Pipeline`.
- Geen secrets, tokens of mailinhoud in logs, tests, fixtures of commits.
- Tests roepen nooit echte externe API's aan (gebruik de fakes).
- Schemawijzigingen altijd via Alembic.
- Frontend moet werken onder een dynamisch Ingress-basispad (geen absolute `/`-paden).
- Kleine commits, Conventional Commits (`feat:`, `fix:`, `docs:` …).
- Na elke fase: lint + typecheck + tests groen, docs bijgewerkt, korte samenvatting.
