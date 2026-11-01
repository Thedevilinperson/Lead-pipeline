# Bedrijfsplatform — Functioneel & technisch ontwerp

> Versie 0.2 — 1 november 2026 (vervangt v0.1 "Home Assistant add-on")
> Werknaam: **Lead Pipeline** (repo-naam); het platform groeit uit tot een
> modulair bedrijfspakket: CRM, klantbeheer, productbeheer, …

---

## 1. Visie en uitgangspunten

**Eindbeeld:** één modulair platform waarin alle bedrijfsonderdelen samenkomen
(relaties, leads & pipelines, mail, klanten, producten, offertes, …), met één
centrale databank op de Home Assistant-server die via meerdere kanalen
raadpleegbaar en bewerkbaar is.

**Eerste versie:** een *eenvoudige, bruikbare* CRM om partners en andere
partijen te contacteren over ons aanbod en dat op te volgen, inclusief
Gmail-threads per lead.

Uitgangspunten:

1. **Databank centraal, apps eromheen.** De databank (PostgreSQL) staat op de
   HA-server. De applicatie is maar één van de kanalen die erop werken.
2. **Modulaire monoliet.** Eén codebase, één API, maar opgedeeld in modules
   met duidelijke grenzen (eigen tabellen, eigen API-routes, eigen schermen).
   Modules aan/uit via configuratie. Geen microservices: te complex voor de start.
3. **Eenvoud eerst.** v1 = kern + CRM + Mail. Al de rest is een latere module.
4. **Mens beslist.** Niets wordt automatisch verzonden; AI doet enkel voorstellen.
5. **Privacy by design.** Enkel relevante mails, data blijft op eigen hardware
   (behalve mailverkeer en — als je dat kiest — AI-aanroepen).

### Niet-doelen v1
Bulkmailing/nieuwsbrieven, facturatie, boekhouding, multi-tenant/SaaS.

---

## 2. Architectuur

```
                          Kanalen (lezen + bewerken)
   ┌───────────┬──────────────┬─────────────┬──────────────┬──────────────┐
   │ Web-app   │ REST API     │ MCP-server  │ Home         │ SQL (alleen- │
   │ (browser, │ (scripts,    │ (Claude     │ Assistant    │ lezen: Excel,│
   │  gsm)     │  n8n, …)     │  Desktop/   │ (sensoren,   │ Power BI,    │
   │           │              │  Code)      │  acties)     │ Metabase)    │
   └─────┬─────┴──────┬───────┴──────┬──────┴──────┬───────┴──────┬───────┘
         │            │              │             │              │
         ▼            ▼              ▼             ▼              │
   ┌──────────────────────────────────────────────────────┐       │
   │  Platform-API (FastAPI) — draait in Docker, waar je   │       │
   │  wil: NUC, NAS, pc of VPS                             │       │
   │  ┌──────── core ────────┐  ┌──────── modules ───────┐ │       │
   │  │ auth · gebruikers ·  │  │ crm · mail · (later:   │ │       │
   │  │ relaties · activi-   │  │ klanten · producten ·  │ │       │
   │  │ teiten · taken · tags│  │ offertes · …)          │ │       │
   │  │ · events · audit     │  └────────────────────────┘ │       │
   │  └──────────────────────┘   worker: mail-sync, AI,    │       │
   │                             herinneringen, webhooks   │       │
   └───────────────────────────┬──────────────────────────┘       │
                               │ TCP 5432 (LAN / VPN)              │
   ┌───────────────────────────▼──────────────────────────────────▼──┐
   │  Home Assistant-server (Intel NUC)                                │
   │  PostgreSQL 16 — databank "platform"                              │
   │   schema's: core · crm · mail · (catalog · sales · …)             │
   │   rollen: platform_app (lezen/schrijven) · platform_readonly      │
   │   nachtelijke pg_dump-back-up                                     │
   └───────────────────────────────────────────────────────────────────┘
          │ IMAP/SMTP (TLS)                    │ HTTPS (optioneel)
          ▼                                    ▼
      Gmail (app-wachtwoord)            AI-provider (Claude) of lokaal (Ollama)
```

### 2.1 De databank op de HA-server

**Waarom PostgreSQL en niet SQLite?** Meerdere kanalen schrijven gelijktijdig en
over het netwerk; SQLite is daar niet voor gemaakt. PostgreSQL geeft daarnaast
schema's per module, JSONB voor eigen velden, full-text search, rollen en
rechten, en is rechtstreeks bruikbaar door BI-tools.

Installatie op de NUC hangt af van hoe Home Assistant draait:

| HA-installatie | PostgreSQL draaien als |
|---|---|
| Home Assistant OS | community-add-on voor PostgreSQL (alleen de *databank* is dan een add-on; de app niet) |
| HA Supervised / Docker / Proxmox | eigen container via `deploy/db/docker-compose.yml` |

Afspraken:
- Poort 5432 alleen op het **LAN of via VPN** (bv. Tailscale/WireGuard) — nooit
  via port forwarding of DynDNS openzetten.
- Rollen: `platform_owner` (migraties), `platform_app` (de API),
  `platform_readonly` (rapportering, Excel, BI). Wachtwoorden niet in Git.
- Back-up: nachtelijke `pg_dump` (14 dagen bewaren) naar de back-upmap van de
  server; herstelprocedure gedocumenteerd en getest.

### 2.2 De platform-API

- Python 3.12 · FastAPI · SQLAlchemy 2 · Alembic · Pydantic v2.
- Draait als Docker-container (`deploy/docker-compose.yml`: `api`, `worker`,
  `web`). Kan op de NUC zelf (als die Docker toelaat), op een NAS, een pc of
  een kleine VPS — het enige wat telt is dat hij de databank bereikt.
- **Worker** (zelfde codebase, ander startcommando) voor achtergrondtaken:
  mail-sync, herinneringen, AI-taken, uitgaande webhooks. Takenwachtrij in
  PostgreSQL zelf (`SELECT … FOR UPDATE SKIP LOCKED`) → geen Redis nodig.

### 2.3 De kanalen

| Kanaal | Wat | Authenticatie |
|---|---|---|
| **Web-app** | React-app, responsive (bruikbaar op gsm) | gebruikersnaam + wachtwoord, sessiecookie |
| **REST API** | volledige CRUD, OpenAPI-documentatie op `/api/docs` | persoonlijke API-tokens met scopes (`read`, `write`, per module) |
| **MCP-server** | Claude Desktop / Claude Code kan relaties en leads opzoeken, notities toevoegen, taken aanmaken, pipeline-overzicht vragen | API-token |
| **Home Assistant** | REST-sensoren (open leads, te beantwoorden, opvolgen vandaag) + webhooks naar HA bij events (bv. antwoord ontvangen → melding op gsm) | API-token (read) + HA-webhook-ID |
| **SQL alleen-lezen** | rapportering in Excel/Power Query, Metabase, Grafana; via stabiele **views** in schema `reporting` | rol `platform_readonly` |
| **Webhooks / n8n** | uitgaande events (`lead.created`, `lead.stage_changed`, `mail.reply_received`, …) | HMAC-handtekening |
| **CSV** | import (met kolomkoppeling, dubbeldetectie) en export | via web-app |

**Regel:** alle *schrijfacties* lopen via de API (validatie, audit-log,
events). Rechtstreeks in de databank schrijven is niet voorzien; lezen mag via
de `reporting`-views.

---

## 3. Modulair ontwerp

### 3.1 Wat is een module?

Een module is een Python-package onder `backend/app/modules/<naam>/` plus een
map `frontend/src/modules/<naam>/`, met:

| Onderdeel | Inhoud |
|---|---|
| `manifest.py` | `key`, naam, versie, `depends_on` (bv. `crm` hangt af van `core`), permissies, instellingen-schema |
| `models.py` | SQLAlchemy-modellen in een eigen PostgreSQL-schema (`crm`, `mail`, …) |
| `migrations/` | eigen Alembic-versiemap (één Alembic-omgeving met `version_locations` per module) |
| `api.py` | FastAPI-router, gemount op `/api/<module>/` |
| `services.py` | businesslogica (de router is dun) |
| `events.py` | welke domein-events de module uitstuurt en waarop ze reageert |
| `jobs.py` | achtergrondtaken voor de worker |
| `mcp_tools.py` | tools die de module aan de MCP-server toevoegt |
| `reporting.sql` | alleen-lezen views voor schema `reporting` |
| frontend | routes, menu-items, en **widgets** voor gedeelde schermen (bv. een tab "Leads" op de relatiefiche) |

Modules praten met elkaar **via services en events**, niet door rechtstreeks
elkaars tabellen te wijzigen. Foreign keys naar `core` mogen; naar andere
modules enkel als `depends_on` dat vastlegt.

Een sjabloonmodule `_template/` maakt een nieuwe module starten triviaal.
Modules aan/uit via `ENABLED_MODULES=crm,mail` (core is altijd aan).

### 3.2 Core (altijd aanwezig)

- **Gebruikers & rollen**: `admin`, `gebruiker`, `alleen-lezen`; permissies per
  module. v1 mag met één gebruiker, maar het model is multi-user.
- **Relaties** (het hart van alles): één tabel `parties` voor organisaties en
  personen, met **rollen** (`lead`, `partner`, `klant`, `leverancier`, …) in
  plaats van aparte tabellen. Een partner die klant wordt, blijft dezelfde
  relatie → CRM, klantbeheer en facturatie delen één bron.
- Contactgegevens (`contact_points`: e-mail, telefoon, website), adressen,
  koppelingen persoon ↔ organisatie (functie).
- **Activiteiten/tijdlijn** (polymorf: kan aan elke entiteit hangen),
  **taken**, **tags**, **eigen velden** (JSONB + definitie per entiteit),
  **bijlagen** (bestanden op schijf, metadata in DB).
- **Events**: outbox-tabel; elke wijziging → event → webhooks, HA, AI, andere modules.
- **Audit-log**: wie wijzigde wat, wanneer, via welk kanaal.
- Instellingen, nummering (bv. klantnummers), API-tokens.

### 3.3 Modulekaart

| Module | Fase | Inhoud |
|---|---|---|
| **core** | v1 | zie hierboven |
| **crm** | v1 | leads/opportuniteiten, pipelines & stappen, kanban, opvolging, Vandaag-scherm, dashboard |
| **mail** | v1 | mailbox-koppeling, threads aan relaties/leads, threadstatus, mailen met sjablonen |
| **ai** | v1.1 | samenvatten, classificeren, concepten, prioriteren, verrijken (§6) |
| **klanten** | v2 | klantfiche: klantnummer, btw-nummer (+ VIES-controle), facturatie-adres, betalingsvoorwaarden, omzetting lead → klant |
| **producten** | v2 | catalogus: SKU, omschrijving, categorie, eenheid, prijs, btw-tarief (21/12/6/0 %), prijslijsten, actief/inactief |
| **offertes** | v3 | offertes met lijnen uit producten, PDF, status, omzetting naar order |
| **orders / verkoop** | v3 | bestellingen, levering, koppeling met webshop/betaalplatform |
| **integraties** | v3+ | o.a. verkoopsplatform-synchronisatie, boekhoudpakket, e-facturatie (Peppol) via een bestaande provider i.p.v. zelf bouwen |
| **projecten / support** | later | tickets, opvolging na verkoop |

---

## 4. Datamodel v1

### Schema `core`
- `users` (id, email, naam, wachtwoord-hash argon2, rol, actief)
- `api_tokens` (id, user_id, naam, hash, scopes, laatst_gebruikt, vervalt_op)
- `parties` (id, kind `organization|person`, display_name, legal_name,
  first_name, last_name, language `nl|fr|en|de`, website, vat_number, notes,
  do_not_contact, consent_basis `gerechtvaardigd_belang|toestemming|bestaande_relatie`,
  custom JSONB, created/updated/archived_at)
- `party_roles` (party_id, role) — lead, partner, klant, leverancier, …
- `party_links` (person_id → organization_id, function, is_primary)
- `contact_points` (party_id, type `email|phone|mobile|website|linkedin`,
  value — e-mail lowercase + uniek, is_primary)
- `addresses` (party_id, type, straat, nr, postcode, gemeente, land)
- `activities` (id, subject_type, subject_id, type `note|call|meeting|email|
  stage_change|system`, title, body, occurred_at, created_by, channel)
- `tasks` (id, subject_type, subject_id, title, due_at, done_at, assignee_id)
- `tags`, `taggings` (polymorf)
- `custom_field_defs` (entity, key, label, type, options)
- `attachments`, `settings`, `event_outbox`, `jobs`, `audit_log`, `webhooks`

### Schema `crm`
- `pipelines` (id, name, is_default)
- `stages` (id, pipeline_id, name, position, probability, kind `open|won|lost`,
  followup_days)
- `leads` (id, title, party_id → organisatie, pipeline_id, stage_id, owner_id,
  source, estimated_value, currency, priority, next_action, next_action_at,
  lost_reason, ai_score, stage_changed_at, …)
- `lead_contacts` (lead_id, person_id, role `beslisser|beïnvloeder|gebruiker`)

Standaardpipeline **"Partnerwerving"**: Te contacteren (5 %) → Eerste contact
(10 %, opvolgen na 5 d) → In gesprek (30 %, 7 d) → Voorstel (50 %, 5 d) →
Onderhandeling (70 %, 3 d) → Partner/gewonnen (100 %, won) → Geen interesse (0 %, lost).

### Schema `mail`
- `mail_accounts` (id, type `gmail_imap|imap|gmail_api`, address, imap/smtp-
  instellingen, versleutelde credentials, sync_state JSONB, status)
- `mail_threads` (id, account_id, provider_thread_id, subject, status,
  last_inbound_at, last_outbound_at, message_count, followup_due_at,
  ai_summary, ai_classification)
- `mail_messages` (id, thread_id, provider_message_id, rfc_message_id,
  direction, from, to, cc, sent_at, snippet, body_text (optioneel), has_attachments)
- `mail_thread_links` (thread_id, subject_type `party|lead`, subject_id, linked_by `auto|label|manual`)
- `mail_templates` (id, name, language, subject, body met `{{variabelen}}`)

### Schema `reporting`
Alleen-lezen views: `v_leads`, `v_pipeline_summary`, `v_open_followups`,
`v_parties`, `v_mail_threads`.

---

## 5. Module Mail (v1)

### 5.1 Koppeling — gewoon Gmail-account, geen Workspace

Standaard **IMAP + SMTP met een Google-app-wachtwoord**:
- Voorwaarde: tweestapsverificatie aan op het Google-account; dan een
  app-wachtwoord aanmaken (16 tekens).
- IMAP `imap.gmail.com:993` (TLS), SMTP `smtp.gmail.com:587` (STARTTLS).
- Voordelen t.o.v. de Gmail API voor een persoonlijk account: geen Google
  Cloud-project, geen OAuth-scherm, en geen koppeling die elke 7 dagen vervalt
  (dat gebeurt bij een OAuth-app in *Testing*-status).
- Gmail-specifieke IMAP-extensies gebruiken: `X-GM-THRID` (thread-ID),
  `X-GM-MSGID`, `X-GM-LABELS`, en `X-GM-RAW` om met Gmail-zoeksyntax te zoeken.
  De map "Alle berichten" opzoeken via het SPECIAL-USE-attribuut `\All`
  (de naam is taalafhankelijk).

**Connector-interface** `MailConnector` met implementaties:
1. `GmailImapConnector` (v1, standaard)
2. `GenericImapConnector` (threading via `Message-ID`/`In-Reply-To`/`References`)
   — voor een latere mailbox op een eigen domein
3. `GmailApiConnector` (later, optioneel; OAuth Desktop-client met
   publicatiestatus *In production*)

Credentials worden versleuteld opgeslagen (Fernet; sleutel uit omgevingsvariabele
`PLATFORM_SECRET_KEY`, nooit in de DB of in Git).

### 5.2 Synchronisatie — enkel relevante mails

- **Niet** de hele mailbox. Enkel threads met e-mailadressen uit
  `core.contact_points`, of met het Gmail-label `Pipeline`.
- Initieel: per gekend adres `X-GM-RAW "from:x OR to:x newer_than:180d"`.
- Incrementeel (worker, elke 5 min, instelbaar): nieuwe UID's sinds de laatst
  geziene UID (bewaard met `UIDVALIDITY`; bij wijziging → hersync laatste 14 dagen).
  Enkel headers ophalen, body alleen voor gematchte berichten.
- Nieuw contactadres toegevoegd → gerichte backfill voor dat adres.
- Koppelen: automatisch op adres-match (relatie + meest recente open lead van
  die relatie; bij twijfel → "te koppelen" in de inbox), via label
  `Pipeline/<lead-nr>`, of manueel.

### 5.3 Threadstatus

| Status | Regel |
|---|---|
| `wacht_op_antwoord` | laatste bericht uitgaand, `followup_due_at` nog niet verstreken |
| `opvolgen` | laatste bericht uitgaand en `followup_due_at` verstreken |
| `antwoord_ontvangen` | laatste bericht inkomend → jij bent aan zet |
| `afgesloten` | manueel, of lead in een `won`/`lost`-stap |
| `afwezig` / `bounce` | herkend via headers (`Auto-Submitted`, `X-Autoreply`, mailer-daemon) |

`followup_due_at = last_outbound_at + followup_days` (van de leadstap,
overschrijfbaar per thread). Statuswijziging → activiteit + event.

### 5.4 Mailen vanuit het platform
- Nieuwe mail of antwoord in thread (`In-Reply-To`, `References`), sjablonen
  met variabelen (`{{person.first_name}}`, `{{organization.name}}`, …).
- Standaard eerst als **concept** (IMAP APPEND naar de Concepten-map) of
  preview; verzenden via SMTP pas na expliciete klik. Gmail bewaart
  SMTP-verzonden mail zelf in Verzonden; de sync pikt ze op.
- `do_not_contact` blokkeert mailen.

---

## 6. AI-integratie (optioneel, uit te schakelen)

Twee manieren, die elkaar aanvullen:

**A. AI in het platform** (module `ai`, provider-abstractie: Claude via de
Anthropic API, of lokaal via Ollama; model per taak instelbaar):

| Functie | Output |
|---|---|
| Threadsamenvatting | 2–4 zinnen + openstaande vragen |
| Antwoordclassificatie | geïnteresseerd / vraag / later / geen interesse / afwezig / doorverwezen + voorgestelde volgende stap |
| Opvolgmail-concept | in de taal van het contact, op basis van een instelbaar bedrijfsprofiel (wie zijn we, wat bieden we aan) |
| Dagprioriteit | score 0–100 + reden, sorteert het Vandaag-scherm |
| Verrijking | gestructureerde velden uit geplakte tekst (website, handtekening, …) |

Gestructureerde output (JSON-schema / tool use), gevalideerd met Pydantic,
bewaard in `ai_insights`; voorstellen met Accepteer/Negeer.

**B. MCP-server** (kanaal): vanuit Claude Desktop of Claude Code met je CRM
praten — "welke partners moet ik deze week opvolgen?", "maak een taak om X te
bellen", "vat de thread met Y samen". Tools per module, met het API-token van
de gebruiker (dus dezelfde rechten en audit-log). Schrijftools vragen in de
client bevestiging.

GDPR: contactgegevens van derden worden verwerkt → opnemen in het
verwerkingsregister/privacybeleid; bij een externe AI-provider de
verwerkersvoorwaarden nakijken. Ollama = alles lokaal.

---

## 7. Gebruikerservaring v1 (simpel en bruikbaar)

Menu: **Vandaag · Pipeline · Relaties · Inbox · Dashboard · Instellingen**

- **Vandaag**: taken vandaag/achterstallig, threads *antwoord ontvangen*,
  threads *opvolgen*. Eén klik naar de juiste fiche.
- **Pipeline**: kanban per pipeline; slepen = stapwijziging (gelogd, stelt
  volgende opvolgdatum voor). Lijstweergave met filters.
- **Relaties**: organisaties en personen, zoeken (Ctrl+K), relatiefiche met
  tijdlijn, leads, mailthreads, taken.
- **Inbox**: gekoppelde threads per status, "te koppelen" threads.
- **Dashboard**: leads per stap, gewogen pipelinewaarde, antwoordratio,
  doorlooptijd.
- **Snel toevoegen**: één formulier "nieuwe relatie + lead" in 20 seconden.

---

## 8. Beveiliging

- Web: wachtwoord (argon2) + sessiecookie (HttpOnly, SameSite=Lax); optioneel
  TOTP later. API/MCP/HA: tokens met scopes, gehasht opgeslagen.
- Toegang van buitenaf enkel via reverse proxy met HTTPS of via VPN.
- Secrets via omgevingsvariabelen / `.env` (niet in Git).
- Geen mailinhoud of secrets in logs. Rate limiting op login.
- Audit-log op alle schrijfacties, met kanaal (`web|api|mcp|ha|worker`).

---

## 9. Repo-structuur (doel)

```
lead-pipeline/
├── backend/
│   ├── app/
│   │   ├── core/          # config, db, auth, parties, activities, events, jobs, audit, module-registry
│   │   ├── modules/
│   │   │   ├── _template/
│   │   │   ├── crm/
│   │   │   ├── mail/
│   │   │   └── ai/
│   │   ├── main.py        # API
│   │   └── worker.py      # achtergrondtaken
│   ├── migrations/        # Alembic, version_locations per module
│   ├── tests/
│   └── pyproject.toml
├── frontend/              # React + TS + Vite + Tailwind
│   └── src/{core,modules/{crm,mail,ai}}/
├── mcp/                   # MCP-server (Python, stdio + streamable HTTP)
├── deploy/
│   ├── docker-compose.yml       # api + worker + web
│   ├── docker-compose.dev.yml   # + lokale postgres + test-mailserver
│   ├── db/                      # init-rollen, back-upscript, compose voor postgres
│   └── homeassistant/           # voorbeeld-YAML: REST-sensoren, webhooks, automatisering
├── docs/
│   ├── ONTWERP.md  BESLISSINGEN.md  INSTALLATIE.md  MODULE_MAKEN.md  MAIL_SETUP.md
├── CLAUDE.md
└── CLAUDE_CODE_PROMPT.md
```

---

## 10. Fasering

| Fase | Inhoud | Klaar als… |
|---|---|---|
| 0 | Skelet: module-systeem, core-auth, DB-verbinding, Alembic per module, React-shell, Docker, CI, dev-omgeving | lege app draait lokaal én tegen de PostgreSQL op de HA-server |
| 1 | Core-relaties + CRM: relaties, leads, pipelines, kanban, tijdlijn, taken, Vandaag, CSV-import | volledige manuele opvolging mogelijk |
| 2 | Mail: Gmail via IMAP/SMTP, sync, threadstatus, koppelen, sjablonen, concepten | threads verschijnen bij leads met juiste status |
| 3 | Kanalen: API-tokens, MCP-server, HA-sensoren & webhooks, `reporting`-views, back-up | CRM bevragen vanuit Claude, melding op gsm bij antwoord |
| 4 | AI-module | voorstellen in de UI, volledig uitschakelbaar |
| 5 | Afwerking v1.0: zoeken, export, NL/EN, docs `MODULE_MAKEN.md` | v1.0 |
| 6+ | Modules klanten, producten, offertes, … | per module eigen mini-ontwerp in `docs/modules/` |

---

## 11. Open vragen

1. Hoe draait Home Assistant op de NUC: HA OS, Supervised, Docker of onder Proxmox?
   (bepaalt hoe PostgreSQL geïnstalleerd wordt, en waar de API-container kan draaien)
2. Waar mag de API-container draaien als dat niet op de NUC kan (NAS, pc, VPS)?
3. Eén gebruiker of meerdere?
4. Mag mailinhoud naar een externe AI-provider, of liever lokaal (Ollama)?
5. Welke pipelinestappen gebruikt het bedrijf echt?
