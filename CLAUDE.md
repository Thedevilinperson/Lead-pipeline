# CLAUDE.md — werkafspraken voor het platform

## Project
Modulair bedrijfsplatform (start: CRM met leads, pipelines en Gmail-opvolging;
later klanten, producten, offertes, …). Modulaire monoliet: FastAPI-backend +
worker + React-frontend + MCP-server. Centrale databank: PostgreSQL op de
Home Assistant-server; het platform is GEEN Home Assistant add-on.
Ontwerp: `docs/ONTWERP.md` (leidend). Bouwplan: `CLAUDE_CODE_PROMPT.md`.
Beslissingen buiten het ontwerp: `docs/BESLISSINGEN.md`.

## Taal
- UI-teksten: Nederlands (België) via i18n, Engels als tweede taal.
- Code, identifiers, commits, code-commentaar: Engels.
- Gebruikersdocumentatie (README, docs/): Nederlands.

## Commando's (na Fase 0)
- `make dev` — volledig platform lokaal (postgres, test-mailserver, fakes, seeddata)
- `make test` — backend (pytest, echte postgres) + frontend (vitest)
- `make lint` — ruff, mypy, eslint, prettier --check
- `make migrate` — Alembic-migraties voor alle actieve modules
- `make build` — Docker-images
- Nieuwe module: kopieer `backend/app/modules/_template/` (zie `docs/MODULE_MAKEN.md`)

## Regels
- Modulegrenzen: elke module eigen PostgreSQL-schema, eigen migraties, eigen
  router. Modules spreken elkaar aan via services/events, nooit door elkaars
  tabellen rechtstreeks te wijzigen.
- Alle schrijfacties via API/services (validatie, audit-log, events).
  Externe tools lezen enkel via schema `reporting`.
- Nooit automatisch mails verzenden; AI wijzigt nooit zelf data.
- Nooit de volledige mailbox synchroniseren; enkel gekende contactadressen of label `Pipeline`.
- Geen secrets, wachtwoorden of mailinhoud in logs, tests, fixtures of commits.
- Tests gebruiken een echte PostgreSQL en fakes voor externe diensten.
- Schemawijzigingen altijd via Alembic.
- v1 eenvoudig houden: liever minder dat goed werkt.
- Kleine commits, Conventional Commits (`feat(crm): …`, `fix(mail): …`, `docs: …`).
- Na elke fase: lint + typecheck + tests groen, docs bijgewerkt, korte samenvatting.
