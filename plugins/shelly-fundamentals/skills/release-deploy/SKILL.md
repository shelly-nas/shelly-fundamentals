---
name: release-deploy
description: Release-, build- en deploypijplijn van shelly-nas apps met release-please, GitHub Actions, de acceptatieomgeving (acc) per PR en de herbruikbare workflows uit shelly-nas/shared-workflows. Gebruik bij het opzetten of aanpassen van .github/workflows (CI, acc, deploy), release-please-config, versiebeheer, rollbacks, GitHub-variabelen en -secrets, of de deploy naar de NAS.
---

# Release en deploy

## De flow

0. **Elke PR naar `main` gaat eerst naar acc**, op `__SUBDOMAIN__-acc.shelly-nas.nl` (zie hieronder).
   Daar bekijk je de wijziging. Pas daarna merge je.
1. Elke push naar `main` draait **release-please**. Dat leest de Conventional Commits sinds de
   laatste tag en houdt een PR **"chore(main): release x.y.z"** bij met de changelog en de
   opgehoogde versies. Op dat moment wordt nog niets gedeployd.
2. **Het mergen van die release-PR is de release.** Dat maakt de tag `vx.y.z` en de GitHub Release.
3. Pas daarna bouwen de images (getagd met `x.y.z` en `latest`) en volgt de deploy met `IMAGE_TAG=x.y.z`.

Een gewone merge naar `main` is dus altijd veilig.
**Rollback:** de images van elke versie blijven in de registry staan, dus er hoeft niets opnieuw
gebouwd te worden. Let op: `deploy.yml` **verwijdert de `.env` na elke deploy**, omdat er geheimen in staan.
Maak daarom op de NAS in `/volume1/docker/__APP__` eerst een `.env` aan met dezelfde waarden als
`env_file_content`, met `IMAGE_TAG` op de eerdere versie. Draai dan `docker compose up -d` en
verwijder de `.env` daarna weer. Migraties worden daarbij **niet** teruggedraaid (zie `database-migraties`).

Draaiende containers houden hun environment, en dankzij `restart: always` komen ze na een reboot
vanzelf terug. Alleen een handmatige `docker compose up` heeft de `.env` nodig.

## Acceptatie (acc)

Elke app heeft **één** acc-omgeving naast productie: `__SUBDOMAIN__-acc.shelly-nas.nl`. Die staat
in een eigen map (`/volume1/docker/__APP__-acc`), met eigen containers (`__APP__-acc-*`),
eigen Traefik-routers (`__SUBDOMAIN__-acc*`) en een eigen database.

- **Wanneer:** `acc.yml` draait bij het openen van een PR naar `main` en bij elke push daarop.
  Er is één acc, dus de laatst gepushte PR staat erop (`concurrency: acc-deploy`). Met *Run workflow*
  zet je een willekeurige branch terug. Release-PR's en PR's van forks gaan niet naar acc.
- **Images:** dezelfde Dockerfiles als productie, getagd `acc-<commit>`. Nooit `latest` of een versienummer.
- **Database:** bij elke deploy volledig opnieuw opgebouwd, uit `init.sql`, `seed.sql` en
  `database/seed-demo.sql`. **Nooit productiedata op acc.** Wat je op acc invoert, is bij de volgende deploy weg.
  De migraties van de PR draaien daarna op die verse database.
- **Workflow:** `shared-workflows/deploy-acc.yml`. Die weigert elke `deploy_directory` die niet op
  `-acc` eindigt, omdat hij de database in die map leegmaakt.
- **Hostnaam:** `<subdomein>-acc` en niet `acc.<subdomein>`. Zo valt acc onder dezelfde wildcard-DNS
  `*.shelly-nas.nl` als productie, zonder extra DNS-regels per app.
- **Goedkeuren:** GitHub laat je je eigen PR niet goedkeuren. Bekijken op acc en dan mergen is de goedkeuring.

`docker-compose.acc.yml` is een kopie van `docker-compose.prod.yml` waarin alleen de namen en de host
verschillen. Een wijziging aan de prod-compose (bijvoorbeeld een nieuwe env-variabele) doe je in beide.

## Templates in deze skillmap

| Template | Doel in de app-repo |
|---|---|
| `templates/deploy.yml` | `.github/workflows/deploy.yml` |
| `templates/acc.yml` | `.github/workflows/acc.yml` (plus `docker-traefik/templates/docker-compose.acc.yml`) |
| `templates/ci.yml` | `.github/workflows/ci.yml` (Node-backend; Python-variant hieronder) |
| `templates/release-please-config.json` | `release-please-config.json` |
| `templates/.release-please-manifest.json` | `.release-please-manifest.json` (begint op `0.0.0`) |

Vervang `__APP__`, `__DB_USER__` en `__DB_NAME__` net als bij `docker-traefik`.

## Herbruikbare workflows (`shelly-nas/shared-workflows`)

Gebruik altijd deze workflows. Kopieer hun stappen niet naar de app. De README van die repo
beschrijft ze volledig. In het kort:

**`build-push.yml`**: één image per aanroep, dus één job per service.
- inputs: `registry`, `image_tag` (volledige tag; meerdere tags gescheiden door een komma),
  `context` (map met de Dockerfile), `runner` (standaard `shelly`)
- secrets: `registry_password`

**`deploy.yml`**
- inputs: `registry`, `deploy_directory` (altijd `/volume1/docker/__APP__`), `compose_file`
  (`docker-compose.prod.yml`), `image_tag`, plus de optionele `runner`, `health_check_delay`, `prune_images` en `prune_age`
- secrets: `registry_username`, `registry_password`, `env_file_content` (regels in de vorm `KEY=VALUE`, worden de `.env` op de NAS)

**`deploy-acc.yml`**
- inputs: `registry`, `deploy_directory` (altijd `/volume1/docker/__APP__-acc`, moet op `-acc` eindigen),
  `image_tag`, `compose_file` (standaard `docker-compose.acc.yml`), `demo_seed_file`
  (standaard `database/seed-demo.sql`), plus de optionele `db_service`, `runner` en `health_check_delay`
- secrets: dezelfde als `deploy.yml`

Alle drie draaien op de self-hosted runner **`shelly`** op de NAS. Release-please en CI draaien op `ubuntu-latest`.

## Instellingen in de GitHub-repo

| Soort | Naam | Inhoud |
|---|---|---|
| Variable | `REGISTRY_URL` | registry-host |
| Variable | `REGISTRY_USERNAME` | registry-gebruiker |
| Variable | `DB_USER`, `DB_NAME` | databasegebruiker en -naam |
| Variable | `TZ` (optioneel) | standaard `Europe/Amsterdam` |
| Secret | `REGISTRY_PASSWORD` | registry-wachtwoord |
| Secret | `DB_PASSWORD` | databasewachtwoord productie |
| Secret | overige app-geheimen | bijvoorbeeld `ANTHROPIC_API_KEY` voor AI-apps |

Komt er een nieuwe omgevingsvariabele bij de server? Voeg hem dan op **alle** plekken toe:
`environment:` in `docker-compose.prod.yml` en `docker-compose.acc.yml`, `env_file_content` in
`deploy.yml` en `acc.yml`, en `.env.example`.

Geef de repo onder *Settings → Actions → General* **"Allow GitHub Actions to create and approve
pull requests"**, anders kan release-please geen PR openen.

## CI

CI draait op elke push en PR (met `concurrency`, zodat een nieuwere push de lopende run annuleert). Minimaal:
- typecheck en tests van client en server
- een Postgres-servicecontainer waarop `init.sql` en `seed.sql` worden toegepast, en de seed
  nog een keer (dat moet een no-op zijn)
- een controle op dubbele migratienummers, en alle migraties twee keer toepassen

Voor een **Python-backend** vervang je de server-stappen door:

```yaml
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip
          cache-dependency-path: server/requirements.txt
      - run: pip install -r requirements.txt pytest
        working-directory: server
      - run: pytest
        working-directory: server
```
