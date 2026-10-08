---
name: stack-voorkeuren
description: Vaste technologiekeuzes voor shelly-nas apps (React/Vite frontend, Node/Express of Python/FastAPI backend, PostgreSQL). Gebruik bij het opzetten van een nieuwe app, het kiezen van een framework, library, database of packagemanager, of bij twijfel welke taal een backend moet krijgen.
---

# Stackvoorkeuren

Elke app bestaat uit drie onderdelen, elk in een eigen map met een eigen Dockerfile:

| Map | Rol | Standaard |
|---|---|---|
| `client/` | Frontend | React + TypeScript, gebouwd met Vite, geserveerd door nginx |
| `server/` | API | Node 20 + Express + TypeScript, **of** Python als de app iets met AI doet |
| `database/` | Database | PostgreSQL 16 (alpine), schema ingebakken in een eigen image |

## Frontend (`client/`)

- React 18 + TypeScript, Vite als bundler. Geen Create React App.
- UI: MUI (`@mui/material`), thema volgens de skill `design-richtlijnen`.
- Routing: `react-router-dom`. Gedeelde state: Redux Toolkit wanneer nodig, anders gewoon React-state.
- Tests: Vitest (`npm test` = `vitest run`).
- API-aanroepen gaan altijd naar het relatieve pad `/api/...`. Nooit een hostnaam of poort hardcoden:
  lokaal proxyt nginx `/api/` naar de server, in productie routeert Traefik het.
- Packagemanager: **npm**, met `package-lock.json` in git en `npm ci` in Dockerfiles en CI.

## Backend (`server/`)

**Kies Python zodra de app iets met AI doet** (LLM-aanroepen, embeddings, RAG, ML-modellen,
agents). Anders Node.

### Node (standaard)
- Node 20, Express, TypeScript, `pg` voor Postgres. Geen ORM; gewone SQL.
- Build met `tsc` naar `dist/`, start met `node dist/index.js`.
- Tests met de ingebouwde `node --test`.

### Python (AI-apps)
- Python 3.12, FastAPI + uvicorn, `psycopg` (v3) voor Postgres. Geen ORM; gewone SQL.
- Dependencies in `requirements.txt` (gepind), app-code in `server/app/`, entrypoint `app.main:app`.
- Voor de Claude API: de officiële `anthropic` SDK en het nieuwste geschikte Claude-model.
- Tests met `pytest`.

### Voor beide backends
- Luistert op poort **3000** (env `SERVER_PORT`), zodat nginx, Traefik en compose identiek blijven.
- Alle routes onder `/api`. Er is altijd een `GET /api/health` die 200 geeft en de versie meldt;
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
- `release-deploy`: release-please en de GitHub Actions-workflows
- `database-migraties`: wijzigingen aan het schema
- `conventional-commits`: commitberichten
- `design-richtlijnen`: kleuren, typografie en het MUI-thema
