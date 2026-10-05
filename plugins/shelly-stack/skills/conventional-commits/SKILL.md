---
name: conventional-commits
description: Verplicht formaat voor commitberichten en PR-titels in shelly-nas repo's (Conventional Commits, dat de versie via release-please bepaalt). Gebruik bij elke git commit, squash-merge of PR-titel.
---

# Commitberichten

De versie wordt automatisch afgeleid uit de commitberichten op `main` (release-please).
**Elk commitbericht moet daarom een Conventional Commit zijn.** Een bericht in een andere vorm
telt niet mee voor de versie en verdwijnt stilletjes uit de changelog.

Formaat: `type(scope): samenvatting`. De scope is optioneel.

| Prefix | Bump | Gebruik voor |
|---|---|---|
| `fix:` | patch | een fout in gedrag die de gebruiker raakte |
| `feat:` | minor | nieuwe functionaliteit |
| `feat!:` / `BREAKING CHANGE:` in de body | major | alles wat een bestaande API, het schema of het deploycontract breekt |
| `perf:` `refactor:` `docs:` `test:` `ci:` `build:` `chore:` | geen | al het andere |

Regels:
- Eén logische wijziging per commit. Een commit met zowel een fix als een feature moet één
  prefix kiezen, en welke dat ook wordt: de release klopt dan niet.
- De samenvatting staat in de gebiedende wijs en begint met een kleine letter:
  `fix: skip duplicate rows on re-import`, niet `Fixed duplicate rows.`
- Een schemawijziging gaat samen met de bijbehorende migratie in dezelfde commit.
- Een breaking change krijgt een `!` **en** een uitleg van het migratiepad in de body.
- Bij een squash-merge wordt de PR-titel het commitbericht. Die moet dus ook aan dit formaat voldoen.
- Pas **nooit** handmatig een versienummer aan. Dat doet de release-PR.
