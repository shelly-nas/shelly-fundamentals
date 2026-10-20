---
name: backend-patronen
description: Patronen voor de Fastify-API van shelly-nas apps - routes en JSON-schema-validatie, statuscodes, één foutformaat, paginering, foutafhandeling en logging, met pg en gewone SQL. Gebruik bij het toevoegen of wijzigen van een endpoint, bij foutafhandeling in de server, of bij code die externe API's of de database aanroept.
---

# Backend-patronen (Node + Fastify)

Aanvulling op `stack-voorkeuren` (Fastify, `buildApp()`, poort 3000, alles onder `/api`). Voor een
Python-backend gelden dezelfde regels, met FastAPI en Pydantic in plaats van Fastify-schema's.

## Opbouw

```
server/src/
  app.ts            buildApp(): registreert plugins en routes, start niets
  index.ts          runMigrations(), dan buildApp().listen()
  routes/<resource>.ts   één Fastify-plugin per resource: schema's + handlers
  db/<resource>.ts       SQL-functies; de enige plek waar queries staan
```

Handlers zijn dun: valideren gebeurt door het schema, data komt uit `db/`, de handler vertaalt alleen.

## Routes

- Resources als meervoud, kleine letters, kebab-case: `/api/boodschappen-lijsten/:id/items`.
- Methoden: `GET` lezen, `POST` aanmaken, `PATCH` deels wijzigen, `PUT` volledig vervangen, `DELETE` verwijderen.
- **Elke route heeft een JSON-schema** voor `params`, `querystring` en `body`, en een `response`-schema
  voor 2xx. Het responsschema filtert ook velden weg die niet naar buiten mogen.

```ts
app.post('/api/items', {
  schema: {
    body: { type: 'object', required: ['naam'], additionalProperties: false,
            properties: { naam: { type: 'string', minLength: 1, maxLength: 200 } } },
    response: { 201: ItemSchema },
  },
}, async (req, reply) => reply.code(201).send(await db.maakItem(req.body.naam)));
```

## Statuscodes

| Code | Wanneer |
|---|---|
| 200 / 201 / 204 | gelukt / aangemaakt / gelukt zonder body |
| 400 | ongeldige invoer (Fastify geeft dit automatisch bij een schemafout) |
| 401 / 403 | niet ingelogd / geen rechten |
| 404 | resource bestaat niet |
| 409 | conflict, bijvoorbeeld een unieke waarde die al bestaat (Postgres-code `23505`) |
| 500 | onverwachte fout; nooit voor invoer van de gebruiker |

## Eén foutformaat

Alle fouten hebben dezelfde vorm, zodat de client één foutafhandeling heeft:

```json
{ "error": { "code": "NOT_FOUND", "message": "Item 42 bestaat niet" } }
```

Zet één `app.setErrorHandler()` in `buildApp()`. Die vertaalt bekende fouten (een eigen
`HttpError(status, code, message)`, schemafouten, Postgres `23505`) naar dit formaat, en logt
onbekende fouten volledig, maar stuurt naar buiten alleen `500` met een algemene melding, zonder stacktrace of SQL.

## Foutafhandeling

- Geen lege `catch`, en geen `.catch(() => [])` of `return null` die een echte fout verbergt. Vang
  alleen wat je kunt afhandelen, laat de rest doorgaan naar de error handler.
- Elke aanroep naar buiten (een externe API, de Claude API, een scraper) heeft een timeout
  (`AbortSignal.timeout(ms)`) en een duidelijke fout als hij faalt. Herhaal alleen idempotente
  aanroepen, met backoff en een maximum aantal pogingen.
- Geen `await` in een lus voor onafhankelijk werk: gebruik `Promise.all`. Geen `forEach(async ...)`.

## Database (`pg`)

- **Altijd geparametriseerd**: `pool.query('SELECT ... WHERE id = $1', [id])`. Nooit SQL samenstellen met
  template strings of `+`, ook niet voor "veilige" waarden.
- Geen `SELECT *`; noem de kolommen, zodat een nieuwe kolom niet ongemerkt in de API belandt.
- Geen query per item in een lus (N+1): haal in één keer op met `WHERE id = ANY($1)` of een `JOIN`.
- Meerdere schrijfacties die samen moeten slagen in één transactie (`BEGIN`/`COMMIT`/`ROLLBACK` op één
  client uit de pool). Houd de transactie kort: geen externe API-aanroep terwijl hij openstaat.
- Paginering: `LIMIT`/`OFFSET` volstaat voor kleine tabellen; bij grote of groeiende tabellen een cursor
  (`WHERE id > $1 ORDER BY id LIMIT $2`).
- Schema, typen en indexen: skill `database-migraties`.

## Logging en configuratie

- Gebruik de logger van Fastify (`req.log`, `app.log`), geen `console.log` in servercode.
- Log nooit wachtwoorden, tokens, API-sleutels of volledige request-bodies met persoonsgegevens.
- Lees `process.env` op één plek bij het opstarten, controleer verplichte waarden en stop met een
  duidelijke melding als er een ontbreekt.
