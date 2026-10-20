---
name: verificatie
description: Controleer je eigen werk in een shelly-nas app voordat je zegt dat het klaar is - build, typecheck, lint en tests met coverage van client en server, plus een korte diff-controle. Gebruik na elke codewijziging, voor een commit of PR, en wanneer gevraagd wordt of iets werkt.
---

# Verificatie

Een wijziging is pas klaar als hij aantoonbaar werkt. Claim nooit dat iets slaagt zonder dat je het
commando hebt gedraaid en de uitkomst hebt gezien. CI is de laatste controle, niet de eerste.

## Volgorde

Draai per gewijzigd project (`client/`, `server/`) en stop bij de eerste fout: los die eerst op.

| Stap | Node (`client/`, `server/`) | Python (`server/`) |
|---|---|---|
| 1. Dependencies | `npm ci` (alleen als `node_modules` ontbreekt of `package-lock.json` wijzigde) | `pip install -r requirements.txt` |
| 2. Typecheck | `npx --no-install tsc --noEmit` | - |
| 3. Lint | `npm run lint` (als het script bestaat) | `ruff check .` (als ruff in de repo zit) |
| 4. Tests + coverage | `npm test` (de drempel van 80% zit in de testconfig, zie skill `testen`) | `pytest` |
| 5. Build | `npm run build` | - |

Houd de uitvoer kort, zodat de context schoon blijft, maar lees altijd de exitcode:

```bash
set -o pipefail
npm test 2>&1 | tail -40
```

Raakt de wijziging het schema of de migraties, controleer dan ook wat CI controleert (skill
`database-migraties`): nummering zonder gaten of dubbelen, en elke migratie twee keer toepasbaar.
Zonder Docker of Postgres op de machine: meld dat expliciet in plaats van het over te slaan.

## Diff-controle

Kijk na de groene run nog één keer naar `git diff`:
- Geen onbedoelde wijzigingen (formattering van ongerelateerde bestanden, debugcode, `console.log`).
- Geen geheimen: geen sleutels, tokens of wachtwoorden in code, compose of SQL. Die horen in `.env`.
- Een nieuwe env-variabele staat op alle vijf plekken (skill `release-deploy`).
- Geen afgezwakte lint- of `tsconfig`-regels, geen `// @ts-ignore`, `eslint-disable` of `.skip` om een
  fout weg te werken. Los de oorzaak op; is afzwakken echt nodig, zeg het dan expliciet.

## Rapport

Sluit af met één kort blok, met de echte uitkomsten:

```
Verificatie
  client: typecheck OK · lint OK · tests 42/42, coverage 86% · build OK
  server: typecheck OK · tests 18/18, coverage 81% · build OK
  niet gedraaid: migraties tegen Postgres (geen Docker op deze machine)
```

Faalt iets dat je niet kunt oplossen, zeg dan wat er faalt en waarom. Schrijf geen OK bij een stap die
je niet hebt gedraaid.
