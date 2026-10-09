---
name: docker-traefik
description: Regels en templates voor docker-compose (lokaal, acceptatie en productie), Dockerfiles, nginx en Traefik-labels van shelly-nas apps. Gebruik bij het dockeriseren van een app, het schrijven of aanpassen van docker-compose.yml / docker-compose.prod.yml of een Dockerfile, healthchecks, volumes, of het publiceren van een app op een *.shelly-nas.nl domein.
---

# Docker en Traefik

Elke app heeft **drie** compose-bestanden met verschillende doelen: lokaal, productie en acceptatie (acc). Begin altijd vanuit de
templates in `templates/` in deze skillmap en vervang de placeholders:

| Placeholder | Betekenis | Voorbeeld |
|---|---|---|
| `__APP__` | Naam van de app: images, containers, deploymap | `finance-app` |
| `__SUBDOMAIN__` | Subdomein onder shelly-nas.nl en de naam van de Traefik-router | `finance` |
| `__DB_USER__` / `__DB_NAME__` | Standaardwaarden voor de database | `financier` / `finance` |

| Template | Doel in de app-repo |
|---|---|
| `templates/docker-compose.yml` | `docker-compose.yml` |
| `templates/docker-compose.prod.yml` | `docker-compose.prod.yml` |
| `templates/docker-compose.acc.yml` | `docker-compose.acc.yml` |
| `templates/client/Dockerfile` + `nginx.conf` | `client/` |
| `templates/server-node/Dockerfile` | `server/` (Node-backend) |
| `templates/server-python/Dockerfile` | `server/` (Python/AI-backend) |
| `templates/database/Dockerfile` | `database/` |
| `templates/.env.example` | `.env.example` |
| `templates/.gitignore` | `.gitignore` |

## Lokaal: `docker-compose.yml`

- Bouwt vanuit de broncode (`build: context: ./client` enzovoort). Geen registry, geen Traefik.
- Vaste poorten: client **5173**→80, server **3000**, db **5432**. Draaien er twee apps tegelijk,
  verschuif dan alleen de poort aan de hostkant.
- Database op een **named volume** (`db_data`). `docker compose down -v` geeft een schone start.
- `init.sql`, `seed.sql` en `seed-demo.sql` worden als read-only bind mounts in
  `/docker-entrypoint-initdb.d/` gezet, genummerd `01-`, `02-`, `03-`.
  `seed-demo.sql` is **alleen lokaal** en komt nooit in het database-image.
- `DB_PASSWORD` is verplicht via `${DB_PASSWORD:?...}` uit `.env`.
- De server wacht op `db` met `condition: service_healthy`.

## Productie: `docker-compose.prod.yml`

De deployworkflow kopieert **alleen dit bestand** (als `docker-compose.yaml`) en een `.env` naar
`/volume1/docker/__APP__` op de NAS. Daaruit volgt:

- **Geen `build:` en geen bind mounts naar de repo.** Alles komt uit images:
  `${REGISTRY}/__APP__-<service>:${IMAGE_TAG:?...}`.
- **Pin `IMAGE_TAG` op een versie** en maak hem verplicht (`:?`). Gebruik nooit `latest` in productie:
  een rollback is dan het aanpassen van één regel, en een herstart haalt niet stilletjes een nieuwere build op.
- **Database-image met ingebakken schema** (`database/Dockerfile`). Een bind mount naar
  `./database/init.sql` bestaat op de host niet: Docker maakt dan een lege map aan en Postgres start zonder schema.
- **Data via een bind mount op `./data/postgres`**, niet via een named volume. Zo overleeft de data
  ook `docker compose down -v`. De map moet op de host bestaan en eigendom zijn van uid/gid **70**
  (de postgres-user in het alpine-image).
- `container_name: __APP__-<service>`, `restart: always` en `TZ: ${TZ:-Europe/Amsterdam}` op elke service.
- **Healthchecks op db en server.** De server-healthcheck roept `GET /api/health` aan met het
  runtime-commando dat al in het image zit (`node -e fetch(...)` of `python -c urllib...`). Installeer geen curl alleen daarvoor.
- **Geen `ports:` in productie.** Alleen Traefik praat met de containers.

## Acceptatie: `docker-compose.acc.yml`

Een kopie van `docker-compose.prod.yml` waarin **alleen** dit verschilt:
- `container_name: __APP__-acc-<service>`
- de routers en services `__SUBDOMAIN__-acc` en `__SUBDOMAIN__-acc-api`
- de host `__SUBDOMAIN__-acc.shelly-nas.nl`

De rest (images, healthchecks, netwerken, `./data/postgres`) is gelijk. De map op de NAS is
`/volume1/docker/__APP__-acc`, dus het compose-project en de data staan los van productie.
Houd beide bestanden gelijk. De flow en de database staan in de skill `release-deploy`.

## Traefik

- Traefik draait al op de NAS. Apps sluiten aan op het **externe netwerk `shelly-network`**.
  Services die Traefik moet bereiken staan in `shelly-network` én `default`; de db zit alleen in `default`.
- Entrypoint is **`web`**. Domein: `__SUBDOMAIN__.shelly-nas.nl`.
- Twee routers per app:
  - `__SUBDOMAIN__-api`: `Host(...) && PathPrefix(`/api`)` → server, poort 3000
  - `__SUBDOMAIN__`: `Host(...)` → client (nginx), poort 80
- Router- en servicenamen moeten uniek zijn over **alle** apps op de NAS, ook acc. Gebruik daarom altijd het subdomein als prefix.
- **`/api` loopt altijd via de Traefik-router naar de server, nooit via nginx.** Prod en acc hebben
  allebei een service `server` op `shelly-network`. Als de client-nginx `server` opzoekt, kan hij dus de
  verkeerde stack treffen. De `/api`-proxy in `nginx.conf` is alleen voor lokaal gebruik.
- Labels altijd als lijst van strings tussen dubbele quotes, met backticks rond de hostnaam.

## Client-image

- Multi-stage: `node:22` bouwt met `npm ci && npm run build`, daarna `nginx:alpine` met `dist/`.
- `nginx.conf` doet SPA-fallback (`try_files ... /index.html`) en proxyt `/api/` naar `server:3000`.
  De upstream staat in een variabele met `resolver 127.0.0.11`, zodat nginx ook opstart als de
  server nog niet draait.

## Images bouwen

Images worden gebouwd op de self-hosted runner `shelly`, op de NAS zelf. Voeg dus geen
`--platform` toe, tenzij een base image er echt om vraagt.
