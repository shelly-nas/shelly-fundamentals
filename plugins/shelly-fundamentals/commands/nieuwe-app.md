---
description: Zet een nieuwe shelly-nas app op (React + Postgres + Node of Python, Docker, Traefik, release-please, deploy naar de NAS)
argument-hint: "[app-naam] [subdomein]"
---

Zet in de huidige repo een nieuwe app op volgens de shelly-nas standaarden. Argumenten: $ARGUMENTS

## 1. Gegevens verzamelen

Vraag wat nog ontbreekt in één keer, met voorstellen als standaard:
- **App-naam** (`__APP__`, kebab-case, bijvoorbeeld `finance-app`). Daarmee worden de images, containers en de deploymap `/volume1/docker/__APP__` benoemd.
- **Subdomein** (`__SUBDOMAIN__`, bijvoorbeeld `finance` voor `finance.shelly-nas.nl`). Dit moet uniek zijn op de NAS.
- **Doet de app iets met AI?** Ja → Python/FastAPI-backend, nee → Node/Express-backend (zie de skill `stack-voorkeuren`).
- **Databasegebruiker en -naam** (`__DB_USER__`, `__DB_NAME__`).
- Een zin over wat de app doet.

## 2. Bestanden aanmaken

Kopieer de templates uit de plugin en vervang overal de placeholders. Laad eerst de skills
`stack-voorkeuren`, `docker-traefik`, `release-deploy`, `database-migraties` en `design-richtlijnen`.

| Bron (`${CLAUDE_PLUGIN_ROOT}/...`) | Doel |
|---|---|
| `skills/docker-traefik/templates/docker-compose.yml` | `docker-compose.yml` |
| `skills/docker-traefik/templates/docker-compose.prod.yml` | `docker-compose.prod.yml` (bij Python: de Python-healthcheck aanzetten en `NODE_ENV` weghalen) |
| `skills/docker-traefik/templates/.env.example` | `.env.example` |
| `skills/docker-traefik/templates/.gitignore` | `.gitignore` |
| `skills/docker-traefik/templates/client/*` | `client/Dockerfile`, `client/nginx.conf` |
| `skills/docker-traefik/templates/server-node/Dockerfile` of `server-python/Dockerfile` | `server/Dockerfile` |
| `skills/docker-traefik/templates/database/Dockerfile` | `database/Dockerfile` |
| `skills/release-deploy/templates/deploy.yml` | `.github/workflows/deploy.yml` |
| `skills/release-deploy/templates/ci.yml` | `.github/workflows/ci.yml` (bij Python: de server-stappen vervangen, zie `release-deploy`) |
| `skills/release-deploy/templates/release-please-config.json` | `release-please-config.json` (bij Python: de `extra-files`-regel voor `server/package.json` weghalen) |
| `skills/release-deploy/templates/.release-please-manifest.json` | `.release-please-manifest.json` |
| `skills/design-richtlijnen/theme.ts` | `client/src/theme.ts` |
| `app-template/CLAUDE.md` | `CLAUDE.md` (`__BACKEND__` = "Node 20 + Express + TypeScript" of "Python 3.12 + FastAPI") |
| `app-template/.claude/settings.json` | `.claude/settings.json` |

Bouw daarna een minimale, werkende basis:
- `client/`: Vite + React + TypeScript + MUI met het thema, light/dark-wissel (systeemvoorkeur + keuze in localStorage), de app-shell uit `design-richtlijnen/patterns.md`, één startpagina die
  `/api/health` aanroept, en Vitest met één test. Figtree via `@fontsource/figtree` (400/500/600/700).
- `server/`: `GET /api/health` (geeft `{ status: "ok", version }`), databaseverbinding via de
  `DB_*`-variabelen, een migratierunner (`runMigrations`, zie `database-migraties`) die vóór het luisteren draait,
  een lege `migrations/` met een README, en één test.
- `database/`: `init.sql` (met een eerste tabel als de app er al een nodig heeft), een idempotente
  `seed.sql` en een `seed-demo.sql`.
- `README.md`: wat de app doet, `cp .env.example .env && docker compose up --build`, en de URL's
  `http://localhost:5173` en `https://__SUBDOMAIN__.shelly-nas.nl`.

Gebruik `npm install` om de lockfiles te genereren. Schrijf ze niet met de hand.

## 3. Controleren

- `docker compose config` op beide compose-bestanden (met een tijdelijke `.env` waarin `DB_PASSWORD` en `IMAGE_TAG` staan).
- Als Docker beschikbaar is: `docker compose up --build -d`, wacht tot alles healthy is, controleer
  `curl localhost:5173/api/health`, en ruim daarna op met `docker compose down -v`.
- Zoek met grep naar overgebleven `__APP__`, `__SUBDOMAIN__`, `__DB_USER__`, `__DB_NAME__` en `__BACKEND__`.

## 4. Afronden

Commit als `feat: scaffold __APP__`. Geef de gebruiker daarna deze checklist voor wat alleen zij zelf kunnen doen:
1. GitHub-repo → *Settings → Secrets and variables → Actions*: variables `REGISTRY_URL`, `REGISTRY_USERNAME`,
   `DB_USER`, `DB_NAME`; secrets `REGISTRY_PASSWORD` en `DB_PASSWORD` (plus bijvoorbeeld `ANTHROPIC_API_KEY` voor AI-apps).
2. *Settings → Actions → General*: "Allow GitHub Actions to create and approve pull requests" aanzetten.
3. Op de NAS: `mkdir -p /volume1/docker/__APP__/data/postgres && sudo chown -R 70:70 /volume1/docker/__APP__/data/postgres`.
4. De eerste release-PR van release-please mergen. Dat deployt `__SUBDOMAIN__.shelly-nas.nl`.
