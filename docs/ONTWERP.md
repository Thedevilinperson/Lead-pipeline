# Lead Pipeline — Functioneel & technisch ontwerp

> Versie 0.1 — 31 oktober 2026
> Doel: een lichte, zelf-gehoste CRM voor het opvolgen van leads en partners, met
> Gmail-threadopvolging en optionele AI, die draait als **Home Assistant add-on**
> en zijn databank op de Home Assistant-server bewaart.

---

## 1. Probleem en doel

Het bedrijf wil partners en andere partijen contacteren over het aanbod. Vandaag
gebeurt dat via losse mails en notities, waardoor:

- niet duidelijk is wie al gecontacteerd werd en wie nog moet antwoorden;
- opvolgmails vergeten worden;
- er geen overzicht is van waar elke partij in het proces zit.

**Lead Pipeline** lost dat op met:

1. een databank van organisaties, contactpersonen en leads;
2. één of meer **pipelines** met stappen (kanban-bord);
3. een **Gmail-koppeling** die mailthreads aan leads hangt en per thread de
   status bijhoudt (wacht op antwoord / antwoord ontvangen / opvolgen / afgesloten);
4. **herinneringen** en een "vandaag te doen"-lijst;
5. **AI-hulp** (optioneel): threads samenvatten, antwoorden classificeren,
   opvolgmails voorstellen en leads prioriteren;
6. **Home Assistant-integratie**: sensoren, events en meldingen op je gsm.

### Niet-doelen (v1)

- Geen bulk-mailing / nieuwsbriefsoftware (geen spam, GDPR).
- Geen automatische verzending van mails zonder menselijke bevestiging.
- Geen facturatie of boekhouding.

---

## 2. Architectuur

```
┌──────────────────────── Home Assistant OS (bv. Intel NUC) ────────────────────────┐
│                                                                                   │
│  HA Core  ◄── Supervisor API (sensoren, events, notify) ──┐                       │
│    ▲                                                      │                       │
│    │ Ingress (HA-login, zijbalkpaneel "Lead Pipeline")    │                       │
│    ▼                                                      │                       │
│  ┌──────────────── Add-on container: lead_pipeline ───────┴─────────────────┐     │
│  │  FastAPI (Python 3.12)                                                   │     │
│  │   ├─ REST API  /api/...                                                  │     │
│  │   ├─ statische frontend (React + Vite build)                             │     │
│  │   ├─ Scheduler (APScheduler): Gmail-sync, herinneringen, HA-sensoren     │     │
│  │   ├─ Gmail-service (google-api-python-client)                            │     │
│  │   └─ AI-service (Anthropic SDK, optioneel Ollama)                        │     │
│  │                                                                          │     │
│  │  /data  (persistent volume, zit in HA-backups)                           │     │
│  │   ├─ leadpipeline.db   (SQLite, WAL-modus)                               │     │
│  │   ├─ secrets.key       (Fernet-sleutel voor tokens)                      │     │
│  │   └─ attachments/      (optioneel)                                       │     │
│  └──────────────────────────────────────────────────────────────────────────┘     │
└───────────────────────────────────────────────────────────────────────────────────┘
           │ HTTPS (uitgaand)                         │ HTTPS (uitgaand, optioneel)
           ▼                                          ▼
      Gmail API (Google)                      Anthropic API (Claude)
```

### Waarom een add-on?

- **Databank op de HA-server**: `/data` is persistent en wordt automatisch
  meegenomen in de Home Assistant-back-ups.
- **Geen aparte login nodig**: Ingress gebruikt de HA-authenticatie; de app
  verschijnt als paneel in de zijbalk en is ook via de HA-app/Nabu Casa bereikbaar.
- **Eén installatie**: deze GitHub-repo is meteen een add-on-repository
  (Instellingen → Add-ons → Add-on store → ⋮ → Repositories → URL van deze repo).

> Vereist Home Assistant OS of Supervised. Draait HA als losse Docker-container,
> dan kan dezelfde image via `docker compose` naast HA draaien (zie §10).

### Technologiekeuzes

| Laag            | Keuze                                   | Reden |
|-----------------|-----------------------------------------|-------|
| Backend         | Python 3.12, FastAPI, Pydantic v2       | Snel, getypeerd, goede Google/Anthropic-SDK's |
| ORM / migraties | SQLAlchemy 2.x + Alembic                | Schema-evolutie zonder dataverlies |
| Databank        | SQLite (WAL) in `/data`                 | Nul beheer, zit in HA-back-ups; MariaDB-add-on als later optie |
| Scheduler       | APScheduler (in-process)                | Geen extra container nodig |
| Frontend        | React 18 + TypeScript + Vite + Tailwind | Kanban, tabellen, snelle UI |
| Drag & drop     | dnd-kit                                 | Kanbanbord |
| Data fetching   | TanStack Query                          | Caching, optimistic updates |
| Gmail           | Gmail API (OAuth 2.0)                   | Threads, labels, concepten, verzenden |
| AI              | Anthropic SDK (Claude), provider-abstractie | Optioneel lokaal model via Ollama |
| Tests           | pytest, httpx, Vitest                   | |

---

## 3. Datamodel

```
organizations ─┬─< contacts >─┬─< lead_contacts >─── leads ──< activities
               │              │                        │  │
               └──────────────┴──< email_addresses     │  └──< email_threads ──< email_messages
                                                       │
pipelines ──< stages ──────────────────────────────────┘
tags >──< taggings (polymorf: organization / contact / lead)
templates · ai_insights · settings · sync_state · audit_log
```

### Tabellen (kern)

**organizations** — `id, name, type (partner|klant|leverancier|overheid|andere),
website, sector, country, city, notes, created_at, updated_at, archived_at`

**contacts** — `id, organization_id?, first_name, last_name, role, phone,
linkedin_url, language (nl|fr|en|de), consent_basis (gerechtvaardigd_belang|
toestemming|bestaande_relatie), do_not_contact (bool), notes, …timestamps`

**email_addresses** — `id, contact_id, email (uniek, lowercase), is_primary`
(een contact kan meerdere adressen hebben; dit is de sleutel voor Gmail-matching)

**pipelines** — `id, name, description, is_default`

**stages** — `id, pipeline_id, name, position, probability (0–100),
kind (open|won|lost), followup_days_default`

Standaardpipeline *"Partnerwerving"*:

| # | Stap                | Kans | Standaard opvolgen na |
|---|---------------------|------|-----------------------|
| 1 | Te contacteren      | 5 %  | –                     |
| 2 | Eerste contact      | 10 % | 5 dagen               |
| 3 | In gesprek          | 30 % | 7 dagen               |
| 4 | Voorstel/aanbod     | 50 % | 5 dagen               |
| 5 | Onderhandeling      | 70 % | 3 dagen               |
| 6 | Gewonnen / Partner  | 100 %| –  (won)              |
| 7 | Verloren / Geen interesse | 0 % | – (lost)        |

**leads** — `id, title, organization_id, pipeline_id, stage_id, owner (HA-user),
source (netwerk|beurs|website|inbound|referral|andere), estimated_value,
currency, priority (laag|normaal|hoog), next_action, next_action_at,
lost_reason, ai_score (0–100)?, stage_changed_at, …timestamps`

**lead_contacts** — `lead_id, contact_id, role (beslisser|beïnvloeder|gebruiker)`

**activities** — `id, lead_id?, contact_id?, type (notitie|telefoon|meeting|
taak|email|stapwijziging|systeem), subject, body, due_at, done_at, created_by`
→ dit is de tijdlijn van een lead. Stapwijzigingen worden automatisch gelogd.

**email_threads** — `id, gmail_thread_id (uniek), lead_id?, subject,
status, last_message_at, last_inbound_at, last_outbound_at, message_count,
followup_due_at, ai_summary, ai_classification, linked_by (auto|label|manueel)`

**email_messages** — `id, thread_id, gmail_message_id (uniek), direction
(inbound|outbound), from_email, to_emails, cc_emails, sent_at, snippet,
body_text (optioneel, configureerbaar), has_attachments`

**templates** — `id, name, language, subject, body` met variabelen
`{{contact.first_name}}`, `{{organization.name}}`, `{{sender.name}}`, …

**ai_insights** — `id, entity_type, entity_id, kind (samenvatting|classificatie|
concept|score), content (JSON), model, created_at`

**sync_state** — `key, value` (bv. Gmail `history_id`, laatste sync-tijd)

**audit_log** — wie wijzigde wat, wanneer (HA-gebruiker via Ingress-header).

---

## 4. Pipelines & opvolging

- **Kanbanbord** per pipeline; slepen = stapwijziging (gelogd in tijdlijn).
- Bij naar een stap slepen wordt `next_action_at` voorgesteld op basis van
  `followup_days_default` van die stap.
- **Lijstweergave** met filters (stap, eigenaar, tag, prioriteit, "geen
  activiteit > X dagen", "wacht op antwoord").
- **Vandaag-scherm**: taken die vandaag/achterstallig zijn, threads met
  status *antwoord ontvangen* (jij bent aan zet) en threads waar opvolging
  verlopen is.
- **Dashboard**: aantal leads per stap, gewogen pipelinewaarde
  (`waarde × kans`), conversie per stap, gemiddelde doorlooptijd, antwoordratio.
- **Import/export**: CSV-import van organisaties/contacten (met kolomkoppeling
  en dubbeldetectie op e-mail/domein), CSV/JSON-export.

---

## 5. Gmail-koppeling

### 5.1 Authenticatie

- Eigen Google Cloud-project → Gmail API activeren → OAuth-client van het type
  **Desktop app**.
- Scopes: `gmail.readonly`, `gmail.send`, `gmail.compose` (concepten),
  `gmail.labels`.
- Flow in de add-on (geen publieke redirect-URL nodig): de app toont de
  Google-autorisatielink met een loopback-redirect (`http://localhost`); na
  toestemming plakt de gebruiker de URL uit de adresbalk terug in de app; de
  backend wisselt de code in (PKCE) voor tokens.
- Refresh-token wordt **versleuteld** (Fernet, sleutel in `/data/secrets.key`)
  opgeslagen; nooit gelogd.
- ⚠️ Een OAuth-app in status *Testing* met gebruikerstype *External* geeft
  refresh-tokens die na ±7 dagen vervallen. Bij een Google Workspace-mailbox:
  kies gebruikerstype **Internal** → geen vervaldatum. De app moet een
  vervallen token netjes detecteren en "Opnieuw koppelen" tonen (+ HA-melding).

### 5.2 Synchronisatie (enkel relevante mails)

Privacy-by-design: **we importeren niet de hele mailbox**, enkel threads met
bekende contactpersonen of met het label `Pipeline`.

1. **Initiële sync** per contact: `users.threads.list` met query
   `from:{email} OR to:{email} newer_than:180d` (periode instelbaar).
2. **Incrementeel**: elke N minuten (standaard 5) `users.history.list` vanaf
   de opgeslagen `history_id`; enkel berichten waarvan een adres matcht met
   `email_addresses` (of label `Pipeline`) worden verwerkt. Bij een te oude
   `history_id` (HTTP 404) → terugvallen op query-sync van de laatste 7 dagen.
3. **Koppelen aan lead**: auto via adres-match (bij meerdere open leads voor
   dezelfde contactpersoon: meest recente open lead, of vraag in UI),
   via Gmail-label `Pipeline/<lead-id>`, of manueel in de app.
4. Pub/Sub-push is bewust **niet** gebruikt (vereist publiek endpoint).

### 5.3 Threadstatus (automatisch)

| Status                 | Regel |
|------------------------|-------|
| `wacht_op_antwoord`    | laatste bericht is uitgaand, `followup_due_at` nog niet verstreken |
| `opvolgen`             | laatste bericht is uitgaand en `followup_due_at` verstreken |
| `antwoord_ontvangen`   | laatste bericht is inkomend → jij bent aan zet |
| `afgesloten`           | manueel, of lead in een `won`/`lost`-stap |
| `bounce` / `afwezig`   | herkend via headers/AI (mailer-daemon, auto-reply) |

`followup_due_at = last_outbound_at + followup_days` (instelling per stap,
overschrijfbaar per thread). Bij statuswijziging → activiteit in tijdlijn +
optioneel HA-event.

### 5.4 Mailen vanuit de app

- Nieuwe mail of antwoord in een bestaande thread (correcte `In-Reply-To`/
  `References`, `threadId`), via sjablonen met variabelen.
- Standaard wordt een **Gmail-concept** aangemaakt; echt verzenden vraagt een
  expliciete klik. Respecteer `do_not_contact`.

---

## 6. AI-integratie (optioneel, uit te schakelen)

Provider-abstractie `AIProvider` met implementaties `AnthropicProvider` en
`OllamaProvider` (lokaal model, data verlaat het netwerk niet). Model per taak
instelbaar (bv. een snel model voor classificatie, een sterker model voor
concepten).

| Functie | Wanneer | Output |
|---|---|---|
| **Threadsamenvatting** | bij nieuw inkomend bericht (opt-in) of op klik | 2–4 zinnen + openstaande vragen |
| **Antwoordclassificatie** | bij inkomend bericht | `geïnteresseerd / vraag / later / geen_interesse / afwezig / doorverwezen` + voorgestelde volgende stap/stapwijziging |
| **Opvolgmail-concept** | op klik | concept in taal van contact, toon van het bedrijf (instelbare "bedrijfsprofiel"-tekst over aanbod) |
| **Prioritering** | dagelijks | score 0–100 + reden; sorteert het Vandaag-scherm |
| **Verrijking** | op klik | gestructureerde velden uit geplakte tekst (website, LinkedIn, e-mailhandtekening) |
| **Vraag het je pipeline** | chatvenster | antwoord op basis van databankquery's (alleen-lezen tools) |

Regels:

- AI **stelt voor, de gebruiker beslist**: geen automatische stapwijzigingen of
  verzendingen; voorstellen verschijnen als "Accepteer / Negeer".
- Gestructureerde output via JSON-schema / tool use; valideren met Pydantic.
- Alleen de nodige context meesturen (geen volledige mailbox); bedragen en
  persoonsgegevens minimaliseren. Log tokenverbruik per functie.
- GDPR: contactgegevens van derden worden verwerkt → vermeld de verwerking in
  het privacybeleid; bij gebruik van een externe AI-provider de
  verwerkersvoorwaarden nakijken. Ollama is het privacyvriendelijke alternatief.

---

## 7. Home Assistant-integratie

Via de Supervisor API (`homeassistant_api: true`, `SUPERVISOR_TOKEN`):

- **Sensoren** (elke 5 min bijgewerkt):
  `sensor.lead_pipeline_open_leads`, `sensor.lead_pipeline_te_beantwoorden`,
  `sensor.lead_pipeline_opvolgen_vandaag`, `sensor.lead_pipeline_pipelinewaarde`
  (gewogen, in EUR), met attributen per stap.
- **Events**: `lead_pipeline_reply_received`, `lead_pipeline_followup_due`,
  `lead_pipeline_stage_changed`, `lead_pipeline_gmail_auth_expired`
  → bruikbaar in automatiseringen (bv. melding op gsm).
- Voorbeeldautomatisering in `DOCS.md`.

---

## 8. Beveiliging

- Alleen bereikbaar via **Ingress** (geen open poort); requests buiten het
  Ingress-IP (`172.30.32.2`) worden geweigerd, behalve in `DEV_MODE`.
- Gebruiker uit Ingress-headers (`X-Remote-User-Id`, `X-Remote-User-Name`) →
  eigenaar & audit-log.
- Secrets (Anthropic-sleutel, Google client secret) via add-on-opties
  (`password`-type); OAuth-tokens versleuteld in de databank.
- Geen mailinhoud in logs. Body-opslag instelbaar (`store_email_bodies`).
- Dagelijkse SQLite-back-up (`VACUUM INTO`) in `/data/backups`, laatste 14
  bewaard — bovenop de HA-back-ups.

---

## 9. Add-on-configuratie (opties)

```yaml
options:
  google_client_id: ""
  google_client_secret: ""
  gmail_sync_interval_minutes: 5
  gmail_initial_sync_days: 180
  store_email_bodies: true
  default_followup_days: 5
  ai_enabled: false
  ai_provider: anthropic        # anthropic | ollama
  anthropic_api_key: ""
  ai_model_fast: claude-haiku-5-5
  ai_model_smart: claude-sonnet-5-5
  ollama_url: "http://homeassistant.local:11434"
  ai_auto_summarize_inbound: false
  ha_sensors_enabled: true
  log_level: info
```

---

## 10. Repo-structuur (doel)

```
lead-pipeline/
├── repository.yaml              # maakt de repo een HA add-on-repository
├── lead_pipeline/               # de add-on
│   ├── config.yaml  build.yaml  Dockerfile  run.sh
│   ├── DOCS.md  CHANGELOG.md  icon.png  logo.png
│   └── translations/nl.yaml en.yaml
├── backend/
│   ├── app/ (main.py, config.py, db.py, models/, schemas/, api/, services/
│   │         gmail/, ai/, ha/, scheduler.py)
│   ├── alembic/   tests/   pyproject.toml
├── frontend/
│   ├── src/ (pages/, components/, api/, hooks/)   package.json  vite.config.ts
├── docker-compose.dev.yml       # lokaal ontwikkelen zonder HA
├── docs/ (ONTWERP.md, GMAIL_SETUP.md)
├── CLAUDE.md                    # werkafspraken voor Claude Code
└── CLAUDE_CODE_PROMPT.md        # de bouwprompt
```

---

## 11. Fasering

| Fase | Inhoud | Klaar als… |
|---|---|---|
| 0 | Skelet: add-on, FastAPI, SQLite, Alembic, React-shell, dev-compose, CI | add-on start in HA en toont lege app via Ingress |
| 1 | CRM-kern: organisaties, contacten, leads, pipelines/stappen, kanban, tijdlijn, taken, Vandaag, CSV-import | volledige manuele opvolging mogelijk |
| 2 | Gmail: OAuth, sync, threadstatus, koppeling aan leads, mailen via concepten/sjablonen | threads verschijnen bij leads met correcte status |
| 3 | HA: sensoren, events, back-ups, dashboard | melding op gsm bij antwoord |
| 4 | AI: samenvatting, classificatie, concepten, prioritering, verrijking | voorstellen in UI, uitschakelbaar |
| 5 | Afwerking: zoekfunctie, export, NL/EN-vertaling, toegankelijkheid, docs | v1.0 release |

---

## 12. Open vragen (voor de eigenaar)

1. Gaat het om een Google Workspace-mailbox (bedrijfsdomein) of een gewoon
   Gmail-account? (bepaalt OAuth-gebruikerstype, zie §5.1)
2. Eén gebruiker of meerdere HA-gebruikers die leads delen?
3. Mag mailinhoud naar een externe AI-provider, of liever lokaal (Ollama)?
4. Welke pipelinestappen gebruikt het bedrijf echt? (standaard in §3 is een voorstel)
5. Draait Home Assistant als HA OS (add-on mogelijk) of als Docker-container?
