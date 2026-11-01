# Bouwprompt voor Claude Code

Start Claude Code in de root van deze repository (`claude` in de map
`Lead-pipeline`) en plak alles uit het codeblok hieronder. Claude Code leest
automatisch `CLAUDE.md` en raadpleegt `docs/ONTWERP.md`.

Tip: zet plan mode aan (Shift+Tab) zodat je het plan voor Fase 0 eerst kunt
nakijken. Geef na elke fase "Ga verder met fase N".

---

```text
Je bent senior software-architect en full-stack engineer. Bouw in deze
repository een MODULAIR BEDRIJFSPLATFORM. Eerste toepassing: een eenvoudige,
bruikbare CRM om partners en andere partijen te contacteren over ons aanbod
en dat op te volgen (leads, pipelines, Gmail-threads met status). Later komen
er modules bij zoals klantbeheer, productbeheer en offertes, dus de
architectuur moet daar vanaf dag één klaar voor zijn — zonder dat v1 daardoor
complex wordt.

## Kernidee
- De databank is centraal: PostgreSQL 16 op mijn Home Assistant-server
  (Intel NUC). Het platform is GEEN Home Assistant add-on. De applicatie is
  één van meerdere kanalen die de data raadplegen en bewerken: web-app,
  REST API, MCP-server (Claude), Home Assistant (sensoren/webhooks), en
  alleen-lezen SQL-views voor rapportering (Excel, Power BI, Metabase).
- Modulaire monoliet: één backend, opgedeeld in modules met harde grenzen.

## Context
- Lees eerst volledig docs/ONTWERP.md (leidend) en CLAUDE.md (werkafspraken).
  Wijk enkel af van het ontwerp bij een concreet probleem; meld dat en noteer
  de beslissing in docs/BESLISSINGEN.md (kort ADR-formaat).
- Gebruikers zijn Nederlandstalig (België). UI via i18n: nl standaard, en als
  tweede taal. Code, identifiers, commits en commentaar in het Engels.
  Gebruikersdocumentatie in het Nederlands.

## Stack (vast)
- Backend: Python 3.12, FastAPI, Pydantic v2, SQLAlchemy 2.x (Mapped[]-stijl),
  Alembic, psycopg 3, argon2-cffi, cryptography (Fernet), httpx,
  imapclient (IMAP), aiosmtplib of smtplib (SMTP), anthropic SDK (optioneel).
  Packaging met uv + pyproject.toml. ruff (lint+format), mypy strict op app/.
- Frontend: React 18 + TypeScript + Vite + Tailwind, TanStack Query,
  React Router, dnd-kit (kanban), react-hook-form + zod, i18next.
  Responsive: ook bruikbaar op gsm.
- MCP-server: Python met de officiële MCP Python SDK, roept de REST API aan
  met een API-token (stdio voor Claude Desktop/Code + streamable HTTP).
- Databank: PostgreSQL 16; per module een eigen PostgreSQL-schema
  (core, crm, mail, ai, reporting).
- Achtergrondtaken: aparte worker-proces uit dezelfde codebase; takenwachtrij
  in PostgreSQL (SELECT ... FOR UPDATE SKIP LOCKED). Geen Redis/Celery.
- Deploy: Docker. deploy/docker-compose.yml (api, worker, web — databank is
  extern via DATABASE_URL), deploy/docker-compose.dev.yml (met lokale
  postgres en een test-mailserver zoals GreenMail of Mailpit),
  deploy/db/ (docker-compose voor postgres op de server, SQL voor rollen,
  back-upscript met pg_dump + 14 dagen retentie, herstelinstructies).
- Tests: pytest met een echte PostgreSQL via testcontainers of de dev-compose
  (geen SQLite-in-plaats-van-Postgres), Vitest voor kritieke frontendlogica.
- CI: GitHub Actions — lint, typecheck, tests (met postgres-service),
  frontend build, docker build.

## Modulesysteem (fundament, Fase 0)
Elke module = backend/app/modules/<key>/ + frontend/src/modules/<key>/ met:
manifest.py (key, naam, versie, depends_on, permissies, instellingen-schema),
models.py (eigen schema), migrations/ (eigen Alembic version_location,
één Alembic-omgeving), api.py (router op /api/<key>/), services.py
(businesslogica), events.py (uitgestuurde en ontvangen domein-events),
jobs.py (worker-taken), mcp_tools.py (MCP-tools), reporting.sql
(alleen-lezen views), en in de frontend: routes, menu-items en widgets voor
gedeelde schermen (bv. een tab op de relatiefiche).
- Een registry laadt modules uit ENABLED_MODULES, controleert depends_on,
  mount routers, registreert jobs, events, MCP-tools en frontend-onderdelen
  (frontend krijgt de lijst actieve modules via /api/core/modules).
- Modules communiceren via services en events, nooit door rechtstreeks
  elkaars tabellen te wijzigen. Foreign keys naar core mogen altijd.
- Lever een sjabloonmodule _template/ en docs/MODULE_MAKEN.md, zodat een
  nieuwe module (bv. "producten") toevoegen een kwestie van kopiëren is.
- Bewijs het systeem met een test die een dummymodule laadt, migreert en
  zijn route aanspreekt.

## Harde vereisten
1. Alle schrijfacties lopen via de API/services (validatie, audit-log,
   events). Externe tools krijgen enkel leestoegang via de rol
   platform_readonly op schema reporting.
2. Databankrollen: platform_owner (migraties), platform_app (runtime),
   platform_readonly (rapportering). Lever het SQL-script om ze aan te maken.
3. Geen enkele mail wordt automatisch verzonden. Sjabloon- en AI-mails worden
   eerst een concept/preview; verzenden vereist een expliciete klik.
   Relaties met do_not_contact = true kunnen niet gemaild worden.
4. Synchroniseer NOOIT de volledige mailbox: enkel threads met e-mailadressen
   van gekende relaties, of met het Gmail-label "Pipeline".
5. Mail voor een gewoon Gmail-account zonder Google Workspace: IMAP + SMTP
   met een Google-app-wachtwoord (vereist tweestapsverificatie). Gebruik de
   Gmail-IMAP-extensies X-GM-THRID, X-GM-MSGID, X-GM-LABELS en X-GM-RAW;
   zoek de "Alle berichten"-map via SPECIAL-USE \All (naam is taalafhankelijk).
   Bouw dit achter een MailConnector-interface met GmailImapConnector (v1),
   GenericImapConnector (threading via Message-ID/In-Reply-To/References)
   en een lege plaats voor een latere GmailApiConnector.
6. Secrets (mailwachtwoorden, AI-sleutels) versleuteld opgeslagen met Fernet;
   de sleutel komt uit de omgevingsvariabele PLATFORM_SECRET_KEY, nooit uit
   de databank of Git. Nooit secrets of mailinhoud loggen of naar de
   frontend sturen.
7. AI stelt enkel voor (Accepteer/Negeer) en wijzigt nooit zelf data. De
   AI-module is volledig uitschakelbaar en het platform werkt 100% zonder.
8. Elke schemawijziging via Alembic. Nooit create_all buiten tests.
9. Audit-log op elke schrijfactie, met gebruiker en kanaal
   (web|api|mcp|ha|worker).
10. Tests roepen nooit echte externe diensten aan (fakes voor mail, AI, HA).

## Werkwijze
- Werk in fasen. Geef vóór Fase 0 een kort plan (structuur, belangrijkste
  beslissingen, risico's) en wacht op mijn akkoord. Voer daarna telkens één
  fase volledig uit: lint, typecheck en tests groen, documentatie bijgewerkt,
  kleine Conventional Commits. Stop met een korte samenvatting en een lijstje
  "dit moet je manueel testen".
- Schrijf tests mee, niet achteraf.
- Externe diensten achter interfaces (MailConnector, AIProvider, HAClient)
  met fakes, zodat `make dev` het volledige platform lokaal draait met
  seeddata, zonder Gmail-, HA- of AI-account.
- Houd v1 eenvoudig: liever minder schermen die goed werken dan veel half.
- Twijfel over een productkeuze buiten het ontwerp? Kies de eenvoudigste
  werkende optie, noteer ze in docs/BESLISSINGEN.md en ga door.

## Fasen

### Fase 0 — Fundament
Repo-structuur uit docs/ONTWERP.md §9. Config via pydantic-settings
(DATABASE_URL, PLATFORM_SECRET_KEY, ENABLED_MODULES, …), DB-sessies,
Alembic met version_locations per module, het modulesysteem + _template +
registry-test, core-auth (gebruikers, argon2, sessiecookie, eerste admin via
CLI-commando), audit-log, event-outbox, job-queue + worker-skelet,
/api/core/health en /api/core/modules, React-shell met login en navigatie
(Vandaag, Pipeline, Relaties, Inbox, Dashboard, Instellingen), Dockerfiles,
beide compose-bestanden, deploy/db (rollen-SQL, compose voor postgres,
back-upscript), Makefile (dev, test, lint, migrate, build), CI,
docs/INSTALLATIE.md (PostgreSQL op de HA-server: variant HA OS met een
community-PostgreSQL-add-on en variant Docker; netwerk enkel LAN/VPN).
Klaar als: lokaal draait alles met `make dev`, en met één DATABASE_URL-
wijziging draait de API tegen de PostgreSQL op de server.

### Fase 1 — Relaties + CRM
Core: parties (organisaties en personen in één model met rollen lead,
partner, klant, leverancier), party_links, contact_points (e-mail uniek,
lowercase), addresses, activities (tijdlijn), tasks, tags, custom fields
(JSONB + definities), full-text zoeken (PostgreSQL tsvector, Ctrl+K).
Module crm: pipelines, stages (kans, kind open/won/lost, followup_days),
leads, lead_contacts; seed standaardpipeline "Partnerwerving" uit het ontwerp.
UI: kanban met drag & drop (stapwijziging logt activiteit en stelt
next_action_at voor), lijstweergave met filters, relatiefiche met tijdlijn,
leads en taken, leadfiche, Vandaag-scherm, snel-toevoegen "relatie + lead",
pipelinebeheer, CSV-import (kolomkoppeling, dubbeldetectie op e-mail en
domein) en export, dashboard (leads per stap, gewogen waarde, doorlooptijd).

### Fase 2 — Mail
Module mail volgens docs/ONTWERP.md §5: mailaccount koppelen in
Instellingen (met stap-voor-stap uitleg + docs/MAIL_SETUP.md over
tweestapsverificatie en app-wachtwoord), verbindingstest, initiële en
incrementele sync in de worker (UID + UIDVALIDITY, enkel headers ophalen,
body enkel bij match), backfill bij nieuw contactadres, automatisch en
manueel koppelen aan relatie/lead, label Pipeline/<lead-nr>, threadstatussen
en regels exact zoals §5.3 (incl. auto-reply/bounce-detectie), Inbox-scherm
met statusfilters en "te koppelen", threads in de tijdlijn, mailen met
sjablonen en variabelen: concept via IMAP APPEND naar de Concepten-map,
verzenden via SMTP pas na klik, correct antwoorden in thread.

### Fase 3 — Kanalen
Persoonlijke API-tokens met scopes (beheer in Instellingen), OpenAPI-docs,
MCP-server in mcp/ met tools per module (zoek relatie, toon lead, pipeline-
overzicht, opvolglijst, notitie toevoegen, taak maken, thread samenvatten
zodra AI aan staat) en installatie-uitleg voor Claude Desktop en Claude Code,
Home Assistant: REST-endpoints voor sensoren + uitgaande webhooks bij events,
met voorbeeld-YAML in deploy/homeassistant/ (rest-sensoren, automatisering
"melding bij antwoord ontvangen"), uitgaande webhooks met HMAC voor n8n e.d.,
schema reporting met de views uit het ontwerp, back-up-job.

### Fase 4 — AI
Module ai: AIProvider-interface met AnthropicProvider en OllamaProvider,
model per taak instelbaar, functies uit §6 (samenvatting, classificatie met
voorgestelde volgende stap, opvolgmail-concept op basis van een instelbaar
bedrijfsprofiel, dagprioriteit, verrijking uit geplakte tekst),
gestructureerde output via tool use / JSON-schema met Pydantic-validatie,
resultaten in ai_insights, tokenverbruik zichtbaar in Instellingen,
promptbestanden apart onder versiebeheer, UI-kaarten met Accepteer/Negeer.

### Fase 5 — Afwerking v1.0
NL/EN volledig, lege toestanden, foutmeldingen in mensentaal,
toetsenbordbediening, README met screenshots, CHANGELOG, tag v1.0.0, en
een kort mini-ontwerp in docs/modules/ voor de volgende modules
(klanten, producten) zodat ze als aparte fase gebouwd kunnen worden.

Begin nu: lees docs/ONTWERP.md en CLAUDE.md en geef je plan voor Fase 0.
```
