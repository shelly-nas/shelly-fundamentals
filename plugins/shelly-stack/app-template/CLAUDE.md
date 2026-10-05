# __APP__

Deze app volgt de shelly-nas standaarden uit de plugin **`shelly-stack@shelly-nas`**
(repo `shelly-nas/shelly-stack`). Gebruik de skills daaruit: `stack-voorkeuren`, `docker-traefik`,
`release-deploy`, `database-migraties`, `conventional-commits` en `design-richtlijnen`.

Is de plugin niet geladen, dan gelden in elk geval deze regels:

- **Stack:** React + Vite + TypeScript + MUI (`client/`), __BACKEND__ (`server/`, poort 3000, alles onder
  `/api`, met `GET /api/health`), PostgreSQL 16 (`database/`).
- **Lokaal:** `docker-compose.yml` bouwt vanuit de broncode. **Productie:** `docker-compose.prod.yml`
  gebruikt alleen images met een gepinde `IMAGE_TAG`, achter Traefik op `shelly-network`, op
  `__SUBDOMAIN__.shelly-nas.nl`.
- **Geheimen** staan alleen in `.env` (niet in git) en in GitHub-secrets. Een nieuwe env-variabele
  komt op drie plekken: `docker-compose.prod.yml`, `env_file_content` in `.github/workflows/deploy.yml` en `.env.example`.
- **Schemawijzigingen:** vóór 1.0.0 in `database/init.sql`. Daarna als idempotente, additieve
  migratie `server/migrations/NNN_*.sql`, en `init.sql` wordt meegewerkt.

## Commitberichten

Versiebeheer is automatisch: release-please leidt de volgende versie af uit de commitberichten op
`main`. **Elke commit is daarom een Conventional Commit.**

| Prefix | Bump |
|---|---|
| `fix:` | patch |
| `feat:` | minor |
| `feat!:` / `BREAKING CHANGE:` in de body | major |
| `perf:` `refactor:` `docs:` `test:` `ci:` `build:` `chore:` | geen |

Eén logische wijziging per commit. De samenvatting in de gebiedende wijs en met een kleine letter.
Pas nooit handmatig een versienummer aan.
