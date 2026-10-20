---
name: shelly-review
description: Review van een wijziging in een shelly-nas app (TypeScript, React, Fastify, PostgreSQL) op beveiliging, correctheid, foutafhandeling en de shelly-nas afspraken (env-variabelen, migraties, acc/prod, commits). Gebruik voor een PR, als gevraagd wordt om een review, of in /klaar.
---

# Review

Review de diff ten opzichte van `main` (`git diff origin/main...HEAD`, plus wat nog niet gecommit is).
Lees van elk gewijzigd bestand ook de omliggende code die je nodig hebt om de wijziging te begrijpen,
maar review alleen wat veranderd is.

Meld alleen echte problemen: iets dat fout gaat, onveilig is of een afspraak breekt. Geen
smaakkwesties en geen stijlpunten die lint al afvangt. Twijfel je, verifieer het dan in de code of met
een test voordat je het meldt.

## 1. Kritiek: beveiliging

- SQL samengesteld met strings in plaats van `$1`-parameters.
- Geheimen in code, compose, SQL of tests; of een geheim in een `VITE_*`-variabele (die komt in de bundel).
- Invoer van buiten zonder schema: een route zonder JSON-schema, `additionalProperties` niet uitgezet waar het hoort.
- Een route die data van een gebruiker wijzigt of toont zonder te controleren of dat mag.
- XSS: `dangerouslySetInnerHTML` of `innerHTML` met invoer, of een `href`/`src` uit invoer zonder controle op `javascript:`.
- Gebruikersinvoer in `child_process`, `fs`-paden of `eval`/`new Function`.
- Wachtwoorden, tokens of persoonsgegevens in logs of foutmeldingen naar de client.

## 2. Hoog: correctheid

- **Async:** een promise zonder `await` of `.catch`, `forEach(async ...)`, `await` in een lus voor onafhankelijk werk.
- **Fouten verborgen:** lege `catch`, `.catch(() => [])`, een fallback die een echte fout als "geen data" laat
  lijken, een verloren stacktrace, een externe aanroep zonder timeout.
- **Types:** `any` zonder reden, `as` om een typefout te omzeilen, `!` zonder voorafgaande controle,
  `@ts-ignore`, of een afgezwakte `tsconfig`/lintregel.
- **React:** een hook in een `if` of na een vroege `return`; een ontbrekende dependency in `useEffect`/`useMemo`;
  een effect zonder cleanup (listener, interval, `fetch` zonder `AbortController`); state direct muteren;
  afgeleide waarde in een effect in plaats van tijdens render; `key={index}` in een lijst die kan veranderen.
- **Database:** een query per item in een lus (N+1), meerdere schrijfacties zonder transactie, een
  ontbrekende index op een nieuwe foreign key of filterkolom.
- **Tests:** nieuw of gewijzigd gedrag zonder test, of een test die ook slaagt als de code fout is (skill `testen`).

## 3. Hoog: shelly-nas afspraken

- Een nieuwe env-variabele staat op alle vijf plekken: `docker-compose.prod.yml`, `docker-compose.acc.yml`,
  `env_file_content` in `deploy.yml` en `acc.yml`, en `.env.example` (skill `release-deploy`).
- `docker-compose.acc.yml` loopt gelijk met prod, op namen, host en databaseopslag na.
- Een schemawijziging na de eerste productiedeploy is een idempotente, additieve migratie met de
  volgende vrije `NNN_`, en `init.sql` is in dezelfde commit bijgewerkt (skill `database-migraties`).
- API-aanroepen in de client gaan naar het relatieve pad `/api/...`; `GET /api/health` werkt nog.
- Commitberichten en PR-titel zijn Conventional Commits, met `!` en migratiepad bij een breaking change.
- UI volgt de tokens en componenten van `design-richtlijnen`: geen losse kleuren, en elementen zijn met het
  toetsenbord bereikbaar, met labels bij invoervelden.

## 4. Midden: alleen melden als het merkbaar is

- Een grote library voor iets kleins, of een import die de hele library binnenhaalt.
- Zwaar werk in render dat bij elke render opnieuw gebeurt.
- `console.log`, uitgecommentarieerde code of dode code die met deze wijziging is achtergebleven.

## Rapport

Per bevinding één regel, de ernstigste eerst:

```
[Kritiek] server/src/routes/items.ts:42 - zoekterm wordt in de SQL geplakt -> gebruik $1
[Hoog]    client/src/pages/Lijst.tsx:18 - fetch in useEffect zonder AbortController -> cleanup toevoegen
```

Sluit af met `Geen blokkerende bevindingen`, of met het aantal kritieke en hoge punten. Kritieke en hoge
punten los je op vóór de PR. Raakt de wijziging authenticatie of betalingen, draai dan ook `/security-review`.
