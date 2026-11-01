# Architectuurbeslissingen

Kort formaat: context → beslissing → gevolgen.

## B-001 — Geen Home Assistant add-on; databank centraal op de HA-server (2026-11-01)
**Context:** de data moet via meerdere kanalen raadpleegbaar en bewerkbaar zijn,
niet enkel via één app in Home Assistant.
**Beslissing:** PostgreSQL draait op de HA-server; de platform-API draait als
losse Docker-container (waar ook) en is één van de kanalen naast REST, MCP,
Home Assistant en alleen-lezen SQL.
**Gevolgen:** databank moet bereikbaar zijn op LAN/VPN; aparte rollen en back-up nodig.

## B-002 — PostgreSQL i.p.v. SQLite (2026-11-01)
**Context:** gelijktijdige toegang over het netwerk, meerdere modules, BI-tools.
**Beslissing:** PostgreSQL 16 met een schema per module.
**Gevolgen:** iets meer beheer; tests draaien tegen echte PostgreSQL.

## B-003 — Modulaire monoliet (2026-11-01)
**Context:** het platform moet uitgroeien tot CRM + klanten + producten + …,
maar v1 moet eenvoudig blijven.
**Beslissing:** één codebase met modules (eigen schema, migraties, router,
frontend-onderdelen), aan/uit via configuratie. Geen microservices.
**Gevolgen:** module-afspraken strikt bewaken (zie CLAUDE.md).

## B-004 — Gmail via IMAP/SMTP met app-wachtwoord (2026-11-01)
**Context:** gewoon Gmail-account, geen Google Workspace. Een OAuth-app in
*Testing*-status geeft refresh-tokens die na 7 dagen vervallen.
**Beslissing:** standaard IMAP + SMTP met een Google-app-wachtwoord, met
Gmail-IMAP-extensies voor threads en labels; achter een `MailConnector`-
interface zodat een generieke IMAP-mailbox of later de Gmail API kan.
**Gevolgen:** tweestapsverificatie vereist; app-wachtwoord versleuteld bewaren.
