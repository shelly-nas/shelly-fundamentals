# shelly-fundamentals

Centrale standaarden voor apps die op de Shelly NAS draaien, als
[Claude Code-plugin](https://code.claude.com/docs/en/plugins). De repo is tegelijk een plugin-marketplace
(`shelly-nas`) met één plugin (`shelly-fundamentals`).

Waarom skills in plaats van één grote CLAUDE.md: Claude ziet van elke skill alleen de korte
beschrijving en laadt de inhoud pas als die relevant is. De Docker-regels komen er dus bij zodra je
iets met Docker doet, maar niet als je een kleur aanpast.

## Inhoud

```
.claude-plugin/marketplace.json        marketplace "shelly-nas"
plugins/shelly-fundamentals/
├── .claude-plugin/plugin.json
├── skills/
│   ├── stack-voorkeuren/              React/Vite, Node of Python (AI), Postgres, npm
│   ├── docker-traefik/                compose lokaal + prod, Dockerfiles, nginx, Traefik-labels
│   │   └── templates/
│   ├── release-deploy/                release-please, CI, deploy via shared-workflows
│   │   └── templates/
│   ├── database-migraties/            init.sql, seed, genummerde migraties
│   ├── conventional-commits/          commitformaat dat de versie bepaalt
│   └── design-richtlijnen/            kleuren, Reddit Mono, MUI-thema, tokens.css
├── commands/nieuwe-app.md             /nieuwe-app: zet een complete app op
└── app-template/                      CLAUDE.md en .claude/settings.json voor app-repo's
```

## Installeren (eenmalig per machine)

In Claude Code:

```
/plugin marketplace add shelly-nas/shelly-fundamentals
/plugin install shelly-fundamentals@shelly-nas
```

## Koppelen aan een app-repo

Zet [`plugins/shelly-fundamentals/app-template/.claude/settings.json`](plugins/shelly-fundamentals/app-template/.claude/settings.json)
in de app-repo als `.claude/settings.json`. Wie de repo opent, krijgt de marketplace en de plugin
dan voorgesteld. Als vangnet, voor het geval de plugin niet laadt (bijvoorbeeld in een
cloudsessie), staat de kern van de regels ook in
[`app-template/CLAUDE.md`](plugins/shelly-fundamentals/app-template/CLAUDE.md). `/nieuwe-app` zet beide bestanden automatisch neer.

## Nieuwe app

In een lege repo:

```
/nieuwe-app boodschappen-app boodschappen
```

## Wijzigen

Pas een skill of template aan, verhoog `version` in `plugins/shelly-fundamentals/.claude-plugin/plugin.json`
en in `.claude-plugin/marketplace.json`, en push. Haal de wijziging lokaal binnen met
`/plugin marketplace update shelly-nas`.

Bron van de huidige standaarden: `shelly-nas/FinanceApp` (compose, workflows, thema) en
`shelly-nas/shared-workflows`.
