---
name: conventional-commits
description: Verplicht formaat voor commitberichten en PR-titels in shelly-nas repo's (Conventional Commits). Gebruik bij elke git commit, squash-merge of PR-titel.
---

# Commitberichten

Elke merge naar `main` gaat direct naar productie, en de geschiedenis van `main` is het logboek
van wat er wanneer live ging. **Elk commitbericht is daarom een Conventional Commit**, zodat die
geschiedenis in één oogopslag laat zien wat een deploy deed en welke deploy iets brak.

Formaat: `type(scope): samenvatting`. De scope is optioneel.

| Prefix | Gebruik voor |
|---|---|
| `fix:` | een fout in gedrag die de gebruiker raakte |
| `feat:` | nieuwe functionaliteit |
| `feat!:` / `BREAKING CHANGE:` in de body | alles wat een bestaande API, het schema of het deploycontract breekt |
| `perf:` `refactor:` `docs:` `test:` `ci:` `build:` `chore:` | al het andere |

Regels:
- Eén logische wijziging per commit. Een commit met zowel een fix als een feature moet één
  prefix kiezen, en dan beschrijft de geschiedenis hem maar half.
- De samenvatting staat in de gebiedende wijs en begint met een kleine letter:
  `fix: skip duplicate rows on re-import`, niet `Fixed duplicate rows.`
- Een schemawijziging gaat samen met de bijbehorende migratie in dezelfde commit.
- Een breaking change krijgt een `!` **en** een uitleg van het migratiepad in de body.
- Bij een squash-merge wordt de PR-titel het commitbericht. Die moet dus ook aan dit formaat voldoen.
