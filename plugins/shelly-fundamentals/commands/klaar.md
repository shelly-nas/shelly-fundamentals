---
description: Rond een wijziging af - verificatie, review, Conventional Commit en een PR naar main (die eerst op acc komt)
argument-hint: "[optioneel: korte omschrijving voor de PR]"
---

Rond de huidige wijziging af volgens de shelly-nas standaarden. Extra context: $ARGUMENTS

## 1. Verificatie

Volg de skill `verificatie` voor elk gewijzigd project: typecheck, lint, tests met coverage (minimaal 80%,
skill `testen`) en build. Faalt er iets, los het op en begin opnieuw bij deze stap. Lukt dat niet, stop
dan en meld wat er faalt; maak geen PR met falende checks.

## 2. Review

Review de diff met de skill `shelly-review`. Los alle kritieke en hoge bevindingen op en draai daarna
stap 1 opnieuw. Noem bevindingen die je bewust laat staan, met de reden, in de PR-beschrijving.

## 3. Commit

Commit volgens de skill `conventional-commits`: één logische wijziging per commit, schemawijziging en
migratie samen. Staan er al losse WIP-commits op de branch, laat ze dan staan: de PR wordt gesquasht en
de PR-titel wordt het commitbericht op `main`.

## 4. PR

Ben je op `main`, maak dan eerst een branch. Push met `git push -u origin <branch>` en maak de PR met
`gh pr create`:
- **Titel:** een Conventional Commit (`feat: ...`, `fix: ...`), want die komt bij de squash op `main`.
- **Beschrijving:** wat er verandert en waarom, het verificatieblok uit stap 1 met de echte uitkomsten,
  en bij een migratie of breaking change het migratiepad.

Meld daarna de PR-link en dat de wijziging na de acc-deploy te bekijken is op `<subdomein>-acc.shelly-nas.nl`.
Merge niet zelf: een merge naar `main` gaat direct naar productie.
