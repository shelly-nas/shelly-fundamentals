---
name: release-deploy
description: Build- en deploypijplijn van shelly-nas apps: de acceptatieomgeving (acc) per PR, direct naar productie bij een merge naar main, de persistente productiedatabase in een extern Docker-volume en de herbruikbare workflows uit shelly-nas/shared-workflows. Gebruik bij het opzetten of aanpassen van .github/workflows (CI, acc, deploy), rollbacks, GitHub-variabelen en -secrets, of de deploy naar de NAS.
---

# Build en deploy

## De flow

1. **Werk op een feature-branch en open een PR naar `main`.** Elke push op die PR wordt gebouwd en
   gedeployd naar acc, `__SUBDOMAIN__-acc.shelly-nas.nl` (zie hieronder). Daar bekijk je de wijziging.
2. **Mergen is releasen.** Elke push naar `main` bouwt de images, getagd met de commit (en `latest`),
   en deployt ze direct naar productie met `IMAGE_TAG=<commit>`.
3. **Acc gaat weg zodra de PR sluit** (gemerged of niet), en anders na de productiedeploy (vangnet).
4. **De branch gaat weg na de merge.** Zet in elke repo *Settings → General → Automatically delete
   head branches* aan, dan verwijdert GitHub de branch zelf. Er blijven dus alleen `main` en de
   branches van open PR's over. Een gesloten PR die niet gemerged is: verwijder de branch met de hand
   als je hem niet meer nodig hebt. Een verwijderde branch is terug te halen met *Restore branch* op de PR.

Er is geen release-PR en geen versienummer: wat in productie draait is altijd een commit op `main`,
en `GET /api/health` meldt die commit (`APP_VERSION`). Merge dus pas als de PR op acc klopt.

**Opnieuw deployen of terugrollen:** *Actions → deploy.yml → Run workflow*. Kies als ref `main` voor
een redeploy, of een branch of tag op een eerdere commit voor een rollback (of revert de commit en
merge die revert: dat is ook een gewone deploy). Een re-run van mislukte jobs gebruikt de oude versie
van de gedeelde workflows; start na een fix daarin dus een nieuwe run.

Migraties worden bij een rollback **niet** teruggedraaid (zie `database-migraties`). Dat gaat goed
zolang ze additief zijn. `shared-workflows/deploy-production.yml` verwijdert de `.env` op de NAS na elke deploy. Draaiende
containers houden hun environment, en dankzij `restart: always` komen ze na een reboot vanzelf terug.
Alleen een handmatige `docker compose up` op de NAS heeft de `.env` nodig.

**De eerste deploy van een bestaande app:** controleer of `init.sql` sinds de database in productie is
aangemaakt veranderd is zonder migratie. Acc en CI bouwen hun database uit `init.sql` en zien dat
niet. Test een nieuwe migratie tegen een database met het vorige schema.

## Productiedata

De productiedatabase staat in een **extern Docker-volume met een vaste naam**, `__APP__-db-data`
(zie `docker-traefik`). Daardoor:

- maakt de eerste deploy het volume aan (`persistent_volumes` van `deploy-production.yml`), en
  hergebruikt elke volgende deploy, ook elke push naar `main`, hetzelfde volume met dezelfde data;
- verwijdert `docker compose down -v` het niet, want compose beheert een extern volume niet;
- laten de opschoonstappen en de acc-teardown het staan: die verwijderen alleen anonieme volumes
  en volumes met het label van het acc-project;
- hangt de naam niet af van de map of de compose-projectnaam. Weggooien is altijd een bewuste
  `docker volume rm __APP__-db-data`.

Zet dezelfde naam in `docker-compose.prod.yml` (`volumes: db_data: name:`) en in `deploy.yml`
(`persistent_volumes`). Draaide de app al met een named volume, gebruik dan de naam van **dat**
volume (`docker volume ls`), dan gaat de bestaande data zonder kopiëren mee.

Acc heeft geen persistent volume: de database daar wordt bij elke deploy opnieuw opgebouwd.

## Acceptatie (acc)

Elke app heeft **één** acc-omgeving naast productie: `__SUBDOMAIN__-acc.shelly-nas.nl`. Die staat
in een eigen map (`/volume1/docker/__APP__-acc`), met eigen containers (`__APP__-acc-*`),
eigen Traefik-routers (`__SUBDOMAIN__-acc*`) en een eigen database.

- **Wanneer:** `acc.yml` deployt bij het openen van een PR naar `main` en bij elke push daarop, en
  breekt acc af als de PR sluit (zie *Afbreken*).
  Er is één acc, dus de laatst gepushte PR staat erop (`concurrency: acc-deploy`). Met *Run workflow*
  zet je een willekeurige branch terug. PR's van forks gaan niet naar acc.
- **Images:** dezelfde Dockerfiles als productie, getagd `acc-<commit>`. Nooit `latest`.
- **Database:** bij elke deploy volledig opnieuw opgebouwd, uit `init.sql`, `seed.sql` en
  `database/seed-demo.sql`, in de bind mount `./data/postgres` van de acc-map.
  **Nooit productiedata op acc.** Wat je op acc invoert, is bij de volgende deploy weg.
  De migraties van de PR draaien daarna op die verse database.
- **Workflow:** `shared-workflows/deploy-acceptance.yml`. Die weigert elke `deploy_directory` die niet op
  `-acc` eindigt, omdat hij de database in die map leegmaakt.
- **Hostnaam:** `<subdomein>-acc` en niet `acc.<subdomein>`. Zo valt acc onder dezelfde wildcard-DNS
  `*.shelly-nas.nl` als productie, zonder extra DNS-regels per app.
- **Goedkeuren:** GitHub laat je je eigen PR niet goedkeuren. Bekijken op acc en dan mergen is de goedkeuring.
- **Afbreken:** acc gaat weg **zodra de PR erop gemerged of gesloten is**. `acc.yml` draait daarvoor ook
  bij `pull_request: closed` en roept dan `shared-workflows/teardown-acceptance.yml` aan met de
  head-commit van de PR. Die leest de commit uit de `acc-<commit>`-tag van de draaiende acc-containers.
  Is dat de commit van de gesloten PR? Dan worden de acc-containers, -netwerken, -volumes, de hele
  acc-map en de ongebruikte images verwijderd. Staat er inmiddels een andere PR (of een met de hand
  gezette branch) op acc, dan blijft acc staan. Een teardown breekt een lopende acc-deploy niet af:
  `cancel-in-progress` staat uit voor `closed`, dus hij wacht zijn beurt af.
  Na een merge staat de wijziging dus op productie en niet meer op acc.
  **Vangnet:** `deploy-production.yml` krijgt `acc_directory`. Na de prod-deploy breekt die
  acc af als de commit erop in de gedeployde geschiedenis zit (merge commit), of als een gemergde PR
  met die commit erin zit (squash of rebase). Dat vangt een branch die je met *Run workflow* op acc
  hebt gezet. De volgende acc-deploy maakt alles opnieuw aan.

`docker-compose.acc.yml` is een kopie van `docker-compose.prod.yml` waarin alleen de namen, de host
en de databaseopslag verschillen. Een wijziging aan de prod-compose (bijvoorbeeld een nieuwe
env-variabele) doe je in beide.

## Templates in deze skillmap

| Template | Doel in de app-repo |
|---|---|
| `templates/deploy.yml` | `.github/workflows/deploy.yml` |
| `templates/acc.yml` | `.github/workflows/acc.yml` (plus `docker-traefik/templates/docker-compose.acc.yml`) |
| `templates/ci.yml` | `.github/workflows/ci.yml` (Node-backend; roept `shared-workflows/ci-node-postgres.yml` aan) |
| `templates/ci-python.yml` | `.github/workflows/ci.yml` bij een Python-backend (zelfstandig; server-stappen vervangen, zie hieronder) |

Vervang `__APP__`, `__SUBDOMAIN__`, `__DB_USER__` en `__DB_NAME__` net als bij `docker-traefik`.

### Afwijkende repo's

**App in een submap** (bijvoorbeeld `app/`, naast andere inhoud in dezelfde repo). Workflows staan
altijd in `.github/workflows/` in de root van de repo, dus alle paden krijgen de submap erbij:
- `context: ./app/server` (en `./app/client`, `./app/database`) bij `build-push-image.yml`
- `compose_file: app/docker-compose.acc.yml` en `demo_seed_file: app/database/seed-demo.sql`:
  `deploy-acceptance.yml` leest beide relatief aan de root van de checkout
- `cache-dependency-path` en `working-directory` in de CI
- een `paths:`-filter (`'app/**'` plus het workflowbestand zelf) op `pull_request` in `acc.yml` en
  op `push`/`pull_request` in `ci.yml`. Zo gaat een PR die alleen de rest van de repo raakt niet
  naar acc. Een `closed`-event valt onder hetzelfde filter, en dat klopt: een PR die acc nooit
  raakte, heeft ook niets af te breken.

**Geen demodata.** Levert `init.sql` (of `seed.sql`) al alles wat acc nodig heeft, zet dan
`demo_seed_file: ''`. Dan slaat `deploy-acceptance.yml` die stap over. Laat geen leeg bestand achter
alleen om aan de standaardwaarde te voldoen.

**Teardown en concurrency.** Houd deploy en teardown in **één** groep (`acc-deploy`) zoals in
`templates/acc.yml`. Een eigen groep per teardown lijkt netter, maar laat hem dan tegelijk met de
deploy van een andere PR lopen: hij ziet nog de oude containers en haalt de acc-map onder die
deploy weg.

## Herbruikbare workflows (`shelly-nas/shared-workflows`)

Gebruik altijd deze workflows. Kopieer hun stappen niet naar de app. De README van die repo
beschrijft ze volledig. In het kort:

**`build-push-image.yml`**: één image per aanroep, dus één job per service.
- inputs: `registry`, `image_tag` (volledige tag; meerdere tags gescheiden door een komma),
  `context` (map met de Dockerfile), `runner` (standaard `shelly`)
- secrets: `registry_password`

**`deploy-production.yml`**
- inputs: `registry`, `deploy_directory` (altijd `/volume1/docker/__APP__`), `compose_file`
  (`docker-compose.prod.yml`), `image_tag` (de commit), `persistent_volumes` (altijd
  `__APP__-db-data`, zie *Productiedata*), `acc_directory` (altijd `/volume1/docker/__APP__-acc`,
  het vangnet uit *Afbreken* hierboven), plus de optionele `runner`, `health_check_delay`, `prune_images` en `prune_age`
- stopt en start de stack met `--remove-orphans`: containers van services die niet meer in de
  compose staan (bijvoorbeeld een oudere opzet met één app-container) verdwijnen mee
- secrets: `registry_username`, `registry_password`, `env_file_content` (regels in de vorm `KEY=VALUE`, worden de `.env` op de NAS)

**`deploy-acceptance.yml`**
- inputs: `registry`, `deploy_directory` (altijd `/volume1/docker/__APP__-acc`, moet op `-acc` eindigen),
  `image_tag`, `compose_file` (standaard `docker-compose.acc.yml`), `demo_seed_file`
  (standaard `database/seed-demo.sql`), plus de optionele `db_service`, `runner` en `health_check_delay`
- secrets: dezelfde als `deploy-production.yml`

**`teardown-acceptance.yml`**: breekt acc af als de PR erop sluit (zie *Afbreken*).
- inputs: `acc_directory` (altijd `/volume1/docker/__APP__-acc`, moet op `-acc` eindigen),
  `commit` (`github.event.pull_request.head.sha`), plus de optionele `runner`, `prune_images` en `prune_age`
- geen secrets

**`ci-node-postgres.yml`**: de CI hieronder, voor een Node-backend.
- inputs: `db_user`, `db_name`, `seed_check_table` (tabel waarvan het aantal rijen gelijk moet blijven
  als de seed opnieuw draait), `migrate_command` (draait `runMigrations()`), plus de optionele
  `node_version`, `typecheck_projects`, `client_directory`, `server_directory`, `init_sql`, `seed_sql`
  en `migrations_directory`

`build-push-image`, `deploy-production`, `deploy-acceptance` en `teardown-acceptance` draaien op de self-hosted runner **`shelly`** op de NAS.
CI en de acc-check van `deploy-production.yml` draaien op `ubuntu-latest`.

**Opschonen.** `deploy-production.yml` en `deploy-acceptance.yml` ruimen na elke run de NAS op, ook als de deploy mislukt
(`teardown-acceptance.yml` doet hetzelfde na een teardown):
images die geen enkele container gebruikt, losse anonieme volumes en losse build-cache. Images van
draaiende **en** gestopte containers blijven staan, en named volumes worden nooit geprunet. Alles wat
weg is, staat nog in de registry. Een rollback werkt dus gewoon. De productiedatabase staat in een
extern volume en de acc-database in een bind mount (`./data/postgres`); geen van beide raakt een prune. Met `prune_age` (bijv. `168h`) houd je recente images
nog even vast; `prune_images: false` zet het opschonen uit.

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

## CI

CI draait op elke push en PR (met `concurrency`, zodat een nieuwere push de lopende run annuleert).
Bij een Node-backend doet `shared-workflows/ci-node-postgres.yml` dit allemaal; de app geeft alleen de
inputs mee (`templates/ci.yml`). Minimaal:
- typecheck en tests van client en server
- een Postgres-servicecontainer waarop `init.sql` en `seed.sql` worden toegepast, en de seed
  nog een keer (dat moet een no-op zijn)
- een controle op dubbele migratienummers, en alle migraties twee keer toepassen
- coverage van client en server met een minimum (`client_coverage_threshold`, `server_coverage_threshold`,
  standaard 80% voor een nieuwe app; skill `testen`)

**Node-backend in JavaScript, zonder build.** `ci-node-postgres.yml` draait altijd `tsc --noEmit` per
project in `typecheck_projects` (een lege lijst kan niet: een matrix moet minstens één waarde hebben) en
`npm run build` in de server. Een JS-server zonder build-script faalt daarop. Schrijf dan een eigen
`ci.yml` met dezelfde stappen (zie de jobs in `ci-node-postgres.yml`): een clientbuild in plaats van
de typecheck, en geen serverbuild. Een ESM-server roept zijn migraties aan via een dynamische import,
bijvoorbeeld:

```yaml
      - name: Apply pending migrations twice
        working-directory: server
        run: |
          for run in 1 2; do
            node -e "import('./src/db.js').then(async (db) => { await db.migreer(); await db.pool.end(); }).catch((e) => { console.error(e); process.exit(1); })"
          done
```

Gebruik voor de Postgres-service dezelfde major als `database/Dockerfile`, via
`mirror.gcr.io/library/postgres:<versie>-alpine` (de runners van GitHub lopen tegen de pull-limiet van
Docker Hub aan). Wordt een JS-backend de norm, maak dan de typecheck en de build optioneel in
`ci-node-postgres.yml` in plaats van dit per app te kopiëren.

Voor een **Python-backend** is er nog geen gedeelde workflow. Neem `templates/ci-python.yml` en
vervang daarin de server-stappen door:

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
