# Lead Pipeline — modulair bedrijfsplatform

Een zelf-gehost, modulair platform voor het bedrijf. Het start als **eenvoudige
CRM** (leads, pipelines en Gmail-threads opvolgen bij partners en andere
partijen) en groeit uit tot een geïntegreerd pakket met **klantbeheer,
productbeheer, offertes, …**

- **Centrale databank** (PostgreSQL) op de Home Assistant-server.
- **Meerdere kanalen** om data te raadplegen en te bewerken: web-app, REST API,
  MCP-server (Claude), Home Assistant, en alleen-lezen SQL voor rapportering.
- **Gmail-koppeling** via IMAP/SMTP (werkt met een gewoon Gmail-account):
  threads per lead met status *wacht op antwoord / antwoord ontvangen /
  opvolgen / afgesloten*.
- **Optionele AI**: samenvatten, antwoorden classificeren, opvolgmails
  voorstellen, prioriteren — met Claude of lokaal via Ollama.

> **Status:** ontwerpfase. Het platform wordt gebouwd met Claude Code op basis
> van de bouwprompt in deze repo.

## Inhoud

| Bestand | Wat |
|---|---|
| [`docs/ONTWERP.md`](docs/ONTWERP.md) | Architectuur, modulesysteem, datamodel, mail, AI, kanalen, beveiliging, fasering |
| [`docs/BESLISSINGEN.md`](docs/BESLISSINGEN.md) | Belangrijke architectuurkeuzes en waarom |
| [`CLAUDE_CODE_PROMPT.md`](CLAUDE_CODE_PROMPT.md) | De prompt om het platform fase per fase te laten bouwen |
| [`CLAUDE.md`](CLAUDE.md) | Vaste werkafspraken die Claude Code automatisch leest |

## Bouwen met Claude Code

```bash
git clone https://github.com/Thedevilinperson/Lead-pipeline.git
cd Lead-pipeline
claude
```

Plak de prompt uit `CLAUDE_CODE_PROMPT.md`. Claude Code stelt eerst een plan
voor Fase 0 voor en bouwt daarna fase per fase:
fundament → relaties + CRM → mail → kanalen (API, MCP, Home Assistant) → AI → v1.0.
Daarna volgen modules zoals klanten en producten.

## Voor je begint

- Hoe draait Home Assistant op de NUC (HA OS, Docker, Proxmox)? Dat bepaalt
  hoe PostgreSQL geïnstalleerd wordt.
- Zet tweestapsverificatie aan op het Gmail-account (nodig voor het app-wachtwoord).
- Mag mailinhoud naar een externe AI-provider, of liever lokaal?
