---
name: testen
description: Test-driven werken in shelly-nas apps - eerst een falende test, dan de kleinste implementatie, dan refactoren - met minimaal 80% coverage. Vitest + Testing Library voor de client, node --test met app.inject() voor de Fastify-server, pytest voor Python. Gebruik bij elke nieuwe feature, bugfix of refactor, en bij het opzetten of aanpassen van testconfig.
---

# Testen

Elke merge naar `main` gaat direct naar productie. Tests zijn daarom de enige controle tussen een
wijziging en de gebruiker. We werken test-first en houden minimaal **80% coverage** op regels,
branches en functies.

## De cyclus

1. **Rood.** Schrijf eerst de test voor het gewenste gedrag en draai hem. Hij moet falen, en wel om de
   juiste reden (het gedrag ontbreekt), niet door een typefout of kapotte setup. Een test die je niet
   hebt zien falen, telt niet.
2. **Groen.** Schrijf de kleinste implementatie waarmee de test slaagt. Draai dezelfde test opnieuw.
3. **Refactor.** Ruim op terwijl de tests groen blijven. Draai daarna de hele suite met coverage.

Bij een **bugfix** is stap 1 een test die de bug reproduceert. Commit test en fix samen (skill
`conventional-commits`): de test bewijst de fix.

## Wat je test

- Gedrag, geen implementatie: wat de gebruiker of aanroeper ziet, niet interne state of private functies.
- Per wijziging: het normale pad, de randgevallen (leeg, grens, ongeldig) en het foutpad.
- Geen snapshottests voor gedrag, geen tests die alleen mocks controleren.

## Client (`client/`): Vitest + Testing Library

- Zoek elementen zoals een gebruiker ze vindt: `getByRole`, `getByLabelText`, `getByText`. `getByTestId`
  alleen als niets anders werkt.
- Interactie met `@testing-library/user-event`, asynchroon met `findBy...` of `waitFor`, nooit met timeouts.
- Mock `fetch` naar `/api/...` op netwerkniveau (MSW of een gestubde `fetch`), niet de component.
- Pure functies (berekeningen, parsers, formattering) als gewone unittests, zonder render.

**De drempel in CI** staat in `.github/workflows/ci.yml`: `client_coverage_threshold` en
`server_coverage_threshold` van `shared-workflows/ci-node-postgres.yml`. CI meet zelf (Vitest-coverage voor de
client, c8 over `dist/` voor de server), zet de cijfers in de job summary en faalt onder de drempel.

Om lokaal dezelfde controle te hebben: coverage in `vite.config.ts` (`@vitest/coverage-v8` als devDependency):

```ts
test: {
  environment: 'jsdom',
  coverage: {
    provider: 'v8',
    include: ['src/**/*.{ts,tsx}'],
    exclude: ['src/**/*.test.{ts,tsx}', 'src/main.tsx', 'src/**/*.d.ts'],
    thresholds: { lines: 80, branches: 80, functions: 80, statements: 80 },
  },
},
```

met `"test": "vitest run --coverage"` in `package.json`.

## Server (`server/`): node --test + `app.inject()`

- Test routes via `buildApp()` en `app.inject()` (skill `stack-voorkeuren`): echte routing, validatie
  en foutafhandeling, zonder poort. Controleer statuscode én body, ook voor 400 en 404.
- Logica die de database raakt test je tegen een echte Postgres (zoals in CI), niet tegen een mock van `pg`.
- **Meet coverage op de gecompileerde code in `dist/`, niet via `tsx`.** Coverage via `--import tsx`
  telt regels altijd als 100%, dus een drempel daarop zegt niets.

```json
"test": "tsc && node --test --experimental-test-coverage --test-coverage-exclude='dist/**/*.test.js' --test-coverage-lines=80 --test-coverage-branches=80 --test-coverage-functions=80 'dist/**/*.test.js'"
```

Onder de drempel eindigt `node --test` met exitcode 1. CI meet de server zelf met c8 over `dist/`; `npm test`
moet daarvoor de tests op `dist/` draaien, niet via `tsx`.

## Python (`server/`): pytest

`pytest-cov` in `requirements.txt` en in `pytest.ini`:

```ini
[pytest]
addopts = --cov=app --cov-branch --cov-report=term-missing --cov-fail-under=80
```

Test FastAPI-routes met `TestClient`, net als `app.inject()` bij Node.

## Bestaande repo onder de 80%

Zet de drempel niet ineens op 80%, want dan faalt CI meteen voor iedereen. Zet hem eerst op `'0'`: CI meet dan
en rapporteert zonder te falen. Neem daarna het laagste van de vier gemeten percentages, naar beneden afgerond,
als drempel, en verhoog hem bij elke PR die tests toevoegt, tot 80%.
Nieuwe of gewijzigde code test je altijd volgens de cyclus hierboven.
