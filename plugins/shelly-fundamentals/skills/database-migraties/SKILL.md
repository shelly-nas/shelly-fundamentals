---
name: database-migraties
description: Regels voor schemawijzigingen in de PostgreSQL-database van shelly-nas apps (init.sql, seed.sql, genummerde migraties in server/migrations). Gebruik bij het toevoegen of wijzigen van tabellen, kolommen, indexen of referentiedata, of bij het schrijven van een migratie.
---

# Database en migraties

## Bestanden

| Bestand | Rol |
|---|---|
| `database/init.sql` | Het volledige schema van een verse installatie |
| `database/seed.sql` | Referentiedata. **Idempotent** (`ON CONFLICT DO NOTHING`), want hij kan opnieuw draaien |
| `database/seed-demo.sql` | Alleen voor lokaal. Slaat zichzelf over als er al data staat. Komt nooit in het image |
| `server/migrations/NNN_omschrijving.sql` | Wijzigingen op een bestaande database |

De init-scripts draaien **alleen bij de eerste start op een leeg datavolume**, op een bestaand volume nooit meer.

## Zolang de app nog niet in productie draait (nog nergens data)

Pas het schema rechtstreeks aan in `init.sql` en `seed.sql`, en begin daarna schoon:

```bash
docker compose down -v && docker compose up --build
```

`server/migrations/` blijft leeg, met alleen een README die uitlegt waarom.

## Vanaf de eerste productiedeploy

Elke schemawijziging wordt dan een migratie. De server past openstaande migraties bij het
opstarten toe, in bestandsvolgorde, en legt in `schema_migrations` vast wat al gedraaid heeft.

Regels:
- **Werk `init.sql` mee bij in dezelfde commit**, zodat een verse en een gemigreerde installatie hetzelfde schema krijgen.
- **Nummer opeenvolgend, zonder gaten of dubbele nummers** (`001_`, `002_`, ...). CI faalt bij een dubbel nummer.
- **Idempotent**: gebruik `CREATE TABLE IF NOT EXISTS`, `ADD COLUMN IF NOT EXISTS` en `CREATE INDEX IF NOT EXISTS`.
  CI draait elke migratie twee keer.
- **Additief**: voeg kolommen en tabellen toe, maar drop of hernoem geen kolom die de vorige deploy nog
  gebruikt. Een rollback zet een oudere server tegen het nieuwere schema, en die moet dan blijven werken.
- Een destructieve wijziging is een bewuste stap in twee deploys: eerst stopt de code met het gebruiken
  van de kolom, pas in een latere PR verdwijnt de kolom. Markeer het als `feat!:` en beschrijf het migratiepad.
- Migratie en code die hem nodig heeft gaan in **één commit**.

## Implementatie in de server

Een migratierunner is klein; schrijf hem zelf, zonder framework:
1. `CREATE TABLE IF NOT EXISTS schema_migrations (version text PRIMARY KEY, applied_at timestamptz DEFAULT now())`
2. Lees `migrations/*.sql` gesorteerd op naam en sla over wat al in `schema_migrations` staat.
3. Draai elk bestand in een eigen transactie en voeg de versie in dezelfde transactie toe.
4. Exporteer de functie (`runMigrations`), zodat CI hem los kan aanroepen. Roep hem aan vóórdat de HTTP-server gaat luisteren.
