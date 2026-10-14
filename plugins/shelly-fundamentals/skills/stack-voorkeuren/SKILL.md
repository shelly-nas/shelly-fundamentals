---
name: stack-voorkeuren
description: Vaste technologiekeuzes voor shelly-nas apps (React/Vite frontend met gewone CSS, Node/Fastify backend, Python/FastAPI alleen voor lokaal ML, PostgreSQL). Gebruik bij het opzetten van een nieuwe app, het kiezen van een framework, library, database of packagemanager, of bij twijfel welke taal een backend moet krijgen.
---

# Stackvoorkeuren

Elke app bestaat uit drie onderdelen, elk in een eigen map met een eigen Dockerfile:

| Map | Rol | Standaard |
|---|---|---|
| `client/` | Frontend | React + TypeScript, gebouwd met Vite, gewone CSS, geserveerd door nginx |
| `server/` | API | Node 22 + Fastify + TypeScript, **of** Python als de app lokaal ML draait |
| `database/` | Database | PostgreSQL 16 (alpine), schema ingebakken in een eigen image |

## Waarom deze keuzes

Bij deze apps bepaalt de taal de snelheid niet: een request wacht vooral op het netwerk en op Postgres,
een AI-request op het model. Daarom wegen twee dingen zwaarder:

- **Eén taal.** TypeScript in de frontend en de backend: gedeelde types, één toolchain, één CI-template,
  en minder context die Claude moet lezen.
- **Weinig tokens per wijziging.** Talen die Claude in één keer goed schrijft. Rust en Go zijn sneller en
  zuiniger met geheugen, maar kosten meer code en meer compileer-rondes voor hetzelfde endpoint, en de
  frontend blijft toch TypeScript. Kies ze alleen als het geheugen op de NAS echt krap wordt
  (`docker stats`), en dan eerst Go.

## Frontend (`client/`)

- React 18 + TypeScript, Vite als bundler. Geen Create React App.
- UI: **gewone CSS** met `tokens.css` en `components.css` uit de skill `design-richtlijnen`. Geen
  componentbibliotheek: MUI voegt een paar honderd KB JavaScript toe. Neem MUI (met `theme.ts`) alleen
  voor een app die al MUI gebruikt, of die een component nodig heeft dat je niet zelf wilt bouwen
  (datagrid, datumkiezer, autocomplete); importeer dan per component (`@mui/material/Button`).
- Routing: `react-router-dom`. Gedeelde state: gewone React-state en context; Redux Toolkit alleen als
  dat echt niet meer volstaat.
- Houd de bundel klein:
  - laad grote pagina's lazy (`React.lazy` per route);
  - kies kleine libraries (`date-fns` in plaats van `moment`, losse functies in plaats van heel `lodash`);
  - controleer de grootte in de uitvoer van `vite build`.
- Tests: Vitest (`npm test` = `vitest run`).
- API-aanroepen gaan altijd naar het relatieve pad `/api/...`. Nooit een hostnaam of poort hardcoden:
  lokaal proxyt nginx `/api/` naar de server, in productie routeert Traefik het.
- Packagemanager: **npm**, met `package-lock.json` in git en `npm ci` in Dockerfiles en CI.

## Backend (`server/`)

**Node is de standaard, ook voor AI-apps** die de Claude API aanroepen (chat, tools, agents, RAG met
een embedding-API). **Kies Python alleen als de app lokaal ML draait**: eigen modellen, PyTorch,
lokale embeddings, of zware data-analyse met pandas of numpy.

### Node (standaard)
- Node 22 (LTS), **Fastify**, TypeScript, `pg` voor Postgres. Geen ORM; gewone SQL.
  Gebruik in CI, Dockerfiles en lokaal dezelfde major-versie. `node --test` met een glob-patroon werkt pas vanaf Node 21;
  op een oudere Node vindt CI dan geen tests.
- Fastify is 2–3× sneller dan Express en valideert requests met een JSON-schema per route. Een bestaande
  Express-app hoeft niet om.
- Bouw de app in een functie (`buildApp()`) en start hem apart, zodat tests `app.inject()` gebruiken
  zonder poort.
- Build met `tsc` naar `dist/`, start met `node dist/index.js`.
- Tests met de ingebouwde `node --test`.
- Voor de Claude API: de officiële `@anthropic-ai/sdk` en het nieuwste geschikte Claude-model.

### Python (lokaal ML)
- Python 3.12, FastAPI + uvicorn, `psycopg` (v3) voor Postgres. Geen ORM; gewone SQL.
- Dependencies in `requirements.txt` (gepind), app-code in `server/app/`, entrypoint `app.main:app`.
- Voor de Claude API: de officiële `anthropic` SDK.
- Tests met `pytest`.
- Een bestaande Python-app (zoals een chatbot) blijft Python; herschrijven levert geen snelheid op.

### Voor beide backends
- Luistert op poort **3000** (env `SERVER_PORT`), zodat nginx, Traefik en compose identiek blijven.
- Alle routes onder `/api`. Er is altijd een `GET /api/health` die 200 geeft en de versie meldt
  (`APP_VERSION`, de gedeployde commit; lokaal de versie uit `package.json`);
  de healthcheck in productie hangt daarvan af.
- Databaseconfig via `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`.
- Migraties staan in `server/migrations/` en worden bij het opstarten toegepast. Zie de skill `database-migraties`.

## Database

- PostgreSQL 16, image `postgres:16-alpine`. **Pin de major-versie**: met `latest` breekt een major-upgrade een bestaand datavolume.
- Schema in `database/init.sql`, referentiedata in `database/seed.sql` (idempotent),
  demodata alleen voor lokaal in `database/seed-demo.sql`.

## Configuratie en geheimen

- Geen wachtwoorden in compose-bestanden, SQL of code. Gebruik `${DB_PASSWORD:?...}` zodat compose weigert te starten zonder.
- `.env` staat in `.gitignore`. `.env.example` staat in git, met lege waarden voor geheimen.
- Tijdzone: `TZ=Europe/Amsterdam`.

## Gerelateerde skills

- `docker-traefik`: compose-bestanden, Dockerfiles en Traefik-labels
- `release-deploy`: de GitHub Actions-workflows, acc per PR, direct naar productie bij een merge, en de persistente productiedata
- `database-migraties`: wijzigingen aan het schema
- `conventional-commits`: commitberichten
- `design-richtlijnen`: huisstijl (Web App Template), tokens, componenten, schermpatronen en het optionele MUI-thema
