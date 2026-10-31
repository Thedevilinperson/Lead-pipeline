# Lead Pipeline

Zelf-gehoste CRM voor het opvolgen van leads, partners en pipelines — draait als
**Home Assistant add-on**, bewaart zijn databank op je Home Assistant-server en
koppelt met **Gmail** om mailthreads per lead op status op te volgen
(wacht op antwoord / antwoord ontvangen / opvolgen / afgesloten).
Optionele **AI** (Claude of een lokaal model via Ollama) vat threads samen,
classificeert antwoorden, stelt opvolgmails voor en prioriteert je dag.

> **Status:** ontwerpfase. De app wordt gebouwd met Claude Code op basis van
> de bouwprompt in deze repo.

## Inhoud van deze repo

| Bestand | Wat |
|---|---|
| [`docs/ONTWERP.md`](docs/ONTWERP.md) | Functioneel & technisch ontwerp: architectuur, datamodel, Gmail-sync, AI, HA-integratie, beveiliging, fasering |
| [`CLAUDE_CODE_PROMPT.md`](CLAUDE_CODE_PROMPT.md) | De prompt om de app fase per fase te laten bouwen door Claude Code |
| [`CLAUDE.md`](CLAUDE.md) | Vaste werkafspraken die Claude Code automatisch leest |

## Bouwen met Claude Code

```bash
git clone https://github.com/Thedevilinperson/Lead-pipeline.git
cd Lead-pipeline
claude
```

Plak daarna de prompt uit `CLAUDE_CODE_PROMPT.md`. Claude Code stelt eerst een
plan voor Fase 0 voor; na je akkoord bouwt het de fasen één voor één
(skelet → CRM-kern → Gmail → Home Assistant → AI → afwerking) en stopt na elke
fase zodat je kunt testen.

## Installeren in Home Assistant (zodra Fase 0 klaar is)

1. **Instellingen → Add-ons → Add-on store → ⋮ → Repositories**
2. Voeg `https://github.com/Thedevilinperson/Lead-pipeline` toe.
3. Installeer **Lead Pipeline**, vul de opties in en start.
4. Open het paneel **Lead Pipeline** in de zijbalk.

Vereist Home Assistant OS of Supervised.

## Voor je begint — beslis even

- Google Workspace-mailbox of gewoon Gmail-account? (bepaalt hoe lang de
  Gmail-koppeling geldig blijft, zie ontwerp §5.1)
- Mag mailinhoud naar een externe AI-provider, of liever lokaal (Ollama)?
- Welke pipelinestappen gebruikt het bedrijf?
