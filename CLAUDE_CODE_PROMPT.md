# Bouwprompt voor Claude Code

Kopieer alles tussen de lijnen hieronder in Claude Code, gestart in de root van
deze repository (`claude` in de map `lead-pipeline`). Claude Code leest ook
automatisch `CLAUDE.md` en kan `docs/ONTWERP.md` raadplegen.

Tip: start met `/plan` (plan mode) zodat je het plan voor Fase 0 eerst kunt
goedkeuren. Geef na elke fase opnieuw "Ga verder met fase N".

---

```text
Je bent senior full-stack engineer. Bouw in deze repository "Lead Pipeline":
een zelf-gehoste CRM voor lead- en pipelineopvolging die draait als
Home Assistant add-on, zijn SQLite-databank in /data op de Home Assistant-server
bewaart, Gmail-threads koppelt aan leads en hun status opvolgt, en optioneel AI
gebruikt (Claude via de Anthropic API, of lokaal via Ollama).

## Context
- Het volledige functionele en technische ontwerp staat in docs/ONTWERP.md.
  Lees het eerst volledig. Het ontwerp is leidend; wijk er enkel van af als je
  een concreet probleem ziet, en meld dat dan expliciet met een voorstel.
- Werkafspraken staan in CLAUDE.md. Volg ze.
- Gebruikers zijn Nederlandstalig (België). UI-teksten in het Nederlands via
  een i18n-bestand (nl als standaard, en als tweede taal). Code, identifiers,
  commits en code-commentaar in het Engels.

## Stack (vast)
- Backend: Python 3.12, FastAPI, Pydantic v2, SQLAlchemy 2.x (typed, Mapped[]),
  Alembic, APScheduler, google-api-python-client + google-auth-oauthlib,
  anthropic SDK, httpx, cryptography (Fernet). Packaging met uv + pyproject.toml.
  Lint/format: ruff. Types: mypy (strict voor app/services).
- Frontend: React 18 + TypeScript + Vite + Tailwind CSS, TanStack Query,
  React Router, dnd-kit (kanban), react-hook-form + zod. Lint: eslint + prettier.
  De frontend moet werken achter een HA Ingress-pad (dynamisch basispad:
  gebruik relatieve asset-paden, `base: './'` in Vite, en lees het
  X-Ingress-Path-header server-side uit om het in index.html te injecteren).
- Databank: SQLite in /data/leadpipeline.db met WAL en foreign keys aan.
  Configureerbaar pad via env DATA_DIR (standaard /data, lokaal ./.data).
- Add-on: repository.yaml in de root + map lead_pipeline/ met config.yaml
  (ingress: true, ingress_port: 8099, panel_icon: mdi:handshake-outline,
  panel_title: Lead Pipeline, homeassistant_api: true, arch: amd64 + aarch64,
  opties en schema zoals in docs/ONTWERP.md §9), build.yaml met de
  officiële HA Python-base-images, Dockerfile (multi-stage: node-build van de
  frontend, daarna Python-image), run.sh die opties uit /data/options.json
  leest, Alembic-migraties uitvoert en uvicorn start.
- Tests: pytest (+ httpx AsyncClient), Vitest voor kritieke frontendlogica.
- CI: GitHub Actions — lint, typecheck, tests backend en frontend, en een
  docker build van de add-on voor amd64.

## Harde vereisten
1. Data verlaat de HA-server alleen naar Google (Gmail API) en, enkel als AI
   aan staat, naar de gekozen AI-provider. Geen telemetrie.
2. Importeer NIET de volledige mailbox. Synchroniseer enkel threads met
   bekende contact-e-mailadressen of met het Gmail-label "Pipeline".
3. Geen enkele mail wordt automatisch verzonden. AI- en sjabloonmails worden
   een Gmail-concept of een preview; verzenden vereist een expliciete klik.
   Contacten met do_not_contact = true kunnen niet gemaild worden.
4. AI stelt enkel voor (accepteer/negeer); het wijzigt nooit zelf stappen,
   statussen of data. AI-functies zijn volledig uit te schakelen en de app
   werkt 100% zonder AI.
5. OAuth-tokens en API-sleutels: versleuteld opgeslagen (Fernet, sleutel in
   DATA_DIR/secrets.key, aangemaakt bij eerste start met rechten 600), nooit
   gelogd, nooit naar de frontend gestuurd. Geen mailinhoud in logs.
6. Requests zonder geldige Ingress-herkomst (bron-IP 172.30.32.2) worden
   geweigerd, behalve wanneer DEV_MODE=true. Lees de HA-gebruiker uit
   X-Remote-User-Id / X-Remote-User-Name voor eigenaarschap en audit_log.
7. Alle schemawijzigingen via Alembic-migraties; nooit create_all in productie.
8. Gmail-OAuth zonder publieke redirect-URL: OAuth-client type "Desktop app",
   PKCE, loopback redirect http://localhost; de gebruiker plakt de
   doorgestuurde URL terug in de app. Detecteer vervallen/ingetrokken tokens
   (invalid_grant), zet de koppeling op "opnieuw koppelen nodig" en vuur het
   HA-event lead_pipeline_gmail_auth_expired.
9. Respecteer Gmail-quota: batch-requests, exponentiële backoff op 429/5xx,
   incrementele sync via users.history.list met fallback naar query-sync bij
   een verlopen historyId.

## Werkwijze
- Werk in fasen (zie hieronder). Begin elke fase met een kort plan
  (bestanden, beslissingen, risico's) en wacht op mijn akkoord vóór fase 0.
  Daarna: voer een fase volledig uit, laat lint/typecheck/tests slagen, werk
  README/DOCS bij, maak kleine logische commits (Conventional Commits), en
  stop met een korte samenvatting + wat ik manueel moet testen.
- Schrijf tests mee, niet achteraf. Minstens: threadstatus-regels,
  e-mailmatching aan contacten/leads, history-sync incl. fallback,
  sjabloonvariabelen, Ingress-beveiliging, en AI-outputvalidatie (met een
  fake provider — tests mogen nooit echte API's aanroepen).
- Externe diensten achter interfaces (GmailClient, AIProvider, HAClient) met
  fakes voor tests en voor DEV_MODE (seeddata + fake Gmail-threads), zodat de
  volledige app lokaal draait met `docker compose -f docker-compose.dev.yml up`
  zonder Google- of HA-account.
- Twijfel je over een productkeuze die niet in het ontwerp staat? Kies de
  eenvoudigste werkende optie, noteer ze in docs/BESLISSINGEN.md en ga door.

## Fasen

### Fase 0 — Skelet
- Repo-structuur zoals docs/ONTWERP.md §10. repository.yaml, add-on-map,
  Dockerfile, run.sh, backend-app met /api/health, Alembic-basis,
  React-shell met navigatie (Vandaag, Pipeline, Leads, Organisaties,
  Contacten, Inbox, Dashboard, Instellingen), dev-compose, CI, .gitignore,
  .editorconfig, Makefile met dev/test/lint/build.
- Klaar als: add-on installeerbaar via de repo-URL in HA, start, en toont de
  lege app als zijbalkpaneel via Ingress; lokaal draait alles via compose.

### Fase 1 — CRM-kern
- Datamodel uit §3 (organizations, contacts, email_addresses, pipelines,
  stages, leads, lead_contacts, activities, tags/taggings, templates,
  settings, audit_log) + migratie + seed van de standaardpipeline
  "Partnerwerving".
- CRUD-API's met paginering, filteren, sorteren en full-text zoeken
  (SQLite FTS5 op organisaties, contacten, leads, notities).
- UI: kanbanbord met drag & drop (stapwijziging logt automatisch een
  activiteit en stelt next_action_at voor), lijstweergave met filters,
  leaddetail met tijdlijn (notities, telefoon, meeting, taak), organisatie-
  en contactdetail, Vandaag-scherm (achterstallige en vandaag-taken),
  pipelinebeheer (stappen toevoegen/herordenen, kans, opvolgtermijn),
  CSV-import met kolomkoppeling + dubbeldetectie (e-mail, domein), export.
- Dashboard: leads per stap, gewogen pipelinewaarde, conversie, doorlooptijd.

### Fase 2 — Gmail
- Instellingenpagina "Gmail koppelen" met de flow uit vereiste 8 en een
  stap-voor-stap uitleg; schrijf docs/GMAIL_SETUP.md (Google Cloud-project,
  Gmail API activeren, OAuth-consentscherm, Desktop-client, scopes
  gmail.readonly, gmail.send, gmail.compose, gmail.labels, en de waarschuwing
  over 7-dagen-tokens bij Testing/External versus Internal bij Workspace).
- Sync-service: initiële sync per contact, incrementele history-sync volgens
  het ingestelde interval, label "Pipeline" en "Pipeline/<lead-id>",
  automatische koppeling aan leads + manuele (her)koppeling in de UI.
- Threadstatussen en regels exact zoals §5.3, inclusief bounce/auto-reply-
  detectie via headers (Auto-Submitted, X-Autoreply, mailer-daemon).
- Inbox-scherm: alle gekoppelde threads met statusfilters; threads in de
  tijdlijn van een lead; threadweergave met berichten.
- Mailen: nieuwe mail en antwoord-in-thread (juiste threadId, In-Reply-To,
  References), sjablonen met variabelen, standaard als Gmail-concept.

### Fase 3 — Home Assistant
- HAClient via Supervisor API: sensoren en events uit §7, ingeschakeld via
  optie ha_sensors_enabled. Voorbeeldautomatisering (melding bij
  lead_pipeline_reply_received) in lead_pipeline/DOCS.md.
- Dagelijkse SQLite-back-up met VACUUM INTO naar /data/backups (14 bewaren).

### Fase 4 — AI
- AIProvider-interface met AnthropicProvider en OllamaProvider; modelkeuze
  per taak via opties (ai_model_fast, ai_model_smart).
- Functies uit §6: threadsamenvatting, antwoordclassificatie met voorgestelde
  volgende stap, opvolgmail-concept (in de taal van het contact, op basis van
  een instelbaar "bedrijfsprofiel" over ons aanbod), dagelijkse prioritering
  met reden, verrijking uit geplakte tekst, en een alleen-lezen
  "vraag het je pipeline"-chat via tool use op veilige query-functies.
- Gestructureerde output via tool use/JSON-schema, gevalideerd met Pydantic;
  resultaten in ai_insights; tokenverbruik loggen en tonen in Instellingen.
- Promptbestanden apart in backend/app/services/ai/prompts/ (versiebeheer).
- UI: voorstellen als kaarten met Accepteer/Negeer; duidelijk label "AI".

### Fase 5 — Afwerking
- Globale zoekbalk (Ctrl+K), NL/EN, toegankelijkheid (toetsenbord, contrast),
  lege toestanden, foutmeldingen in mensentaal, CHANGELOG, versie 1.0.0,
  README met installatie-instructies en screenshots.

Begin nu: lees docs/ONTWERP.md en CLAUDE.md, en geef je plan voor Fase 0.
```
