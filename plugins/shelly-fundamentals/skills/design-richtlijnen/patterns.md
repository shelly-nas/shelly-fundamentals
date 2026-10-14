# Schermpatronen

Afgeleid uit de schermen in het Web App Template (`reference/web-app-template.html`).

## App-shell (desktop, vanaf `md`)

```
┌──────────────┬──────────────────────────────────────────────┐
│ [A] App name │  Page title                    [Search][CTA] │
│ Workspace    │  Subtekst in muted                           │
│              │                                              │
│ ▣ Overview   │  ┌──────── content in cards ──────────────┐  │
│ □ Products   │  │                                        │  │
│ □ Orders  3  │  └────────────────────────────────────────┘  │
│ □ Reports Soon                                              │
│              │                                              │
│ (JC) Name    │                                              │
│      Admin   │                                              │
└──────────────┴──────────────────────────────────────────────┘
```

- **Sidebar:** basisbreedte 248px, achtergrond `--surface`, rechterrand `--border`, padding 24×16.
  Bovenaan een app-logo plus naam, daaronder nav-items (gap 4, met optionele tellers), onderaan de gebruiker (avatar 32px, naam en rol).
  Groepen staan 32px uit elkaar.
- **Main:** `min-width: 0`, padding 32px, achtergrond `--bg`.
- Responsief met één breakpoint (`md`): vanaf `md` staat de sidebar vast (248px, `position: sticky`,
  `top: 0`, `height: 100vh`, `overflow-y: auto`). Daaronder de mobiele shell hieronder. Laat de
  sidebar nooit boven de content wrappen: dan opent een telefoon op het hele menu in plaats van op de pagina.
- **Paginakop:** Title (24/600) met een muted subtekst (8px gap) links; zoeken en de primaire actie rechts
  (`justify-content: space-between; align-items: flex-end; flex-wrap: wrap`).

## App-shell (mobiel, onder `md`)

```
┌──────────────────────────┐      ┌──────────────┬───────────┐
│ ☰  [A] App name          │      │ [A] App name │░░░░░░░░░░░│
├──────────────────────────┤      │              │░░░░░░░░░░░│
│ Page title               │  ☰→  │ ▣ Overview   │░░ scrim ░░│
│ Subtekst                 │      │ □ Products   │░░░░░░░░░░░│
│ ┌──── content ─────────┐ │      │ □ Orders  3  │░░░░░░░░░░░│
│ └──────────────────────┘ │      │ (JC) Name    │░░░░░░░░░░░│
└──────────────────────────┘      └──────────────┴───────────┘
```

- **Topbalk:** `position: fixed`, 56px hoog, achtergrond `--surface`, onderrand `--border`, padding 0 8px,
  `z-index` van de app bar. Links een menuknop van 44×44 (`aria-label` "Menu openen"), daarnaast het
  app-logo plus naam, dezelfde als bovenin de sidebar.
- **Menu:** de menuknop opent dezelfde navigatie als de sidebar in een drawer (links, breedte
  248 + 32px, max `85vw`): een `<dialog>` met `showModal()` en een scrim via `::backdrop`, of in een
  MUI-app een `Drawer` met `backgroundImage: none`. Een klik op een nav-item navigeert en sluit de drawer;
  Escape of de scrim sluiten hem ook.
- **Main:** padding 16px opzij (24px vanaf `sm`) en bovenin 56 + 24px, zodat de paginakop onder de topbalk begint.
- Bouw de navigatie één keer (logo, items, onderste controls) en hang hem in de sidebar of in de drawer,
  zodat tellers en acties op beide plekken gelijk blijven.

## Lijst / dashboard

- Tabs boven de lijst (All · Active · Drafts · Archived, met tellers).
- Zoeken met een icoon in het veld. Bij nul resultaten: een lege staat met de zoekterm, een uitleg en de knop "Clear search".
- Een grid van kaarten (`card-link`: bij hover een sterkere rand en `--shadow-2`) of een tabel in een kaart.
- "Recent …"-blok: een kaart met een Heading en de link "View all" rechts, daaronder een tabel.
- Kleine gemakken: klik op een ID of SKU om te kopiëren, met een toast als bevestiging.

## Instellingenformulier

- Een Title met de subtekst "Changes apply after you save."
- Elke sectie is een kaart (padding 24, gap 24) met een Heading en een muted subtekst, daaronder de velden.
- Velden in een grid van twee kolommen op desktop, één kolom op mobiel. Optionele velden krijgen "(optional)" in het label.
  Een textarea toont een teller (`42 / 160`).
- Meldingen als een lijst met switches (titel plus beschrijving).
- **Gevarenzone** als laatste kaart: titel, een zin die uitlegt wat er onherroepelijk verdwijnt, en een
  destructieve knop met "…" (*Delete workspace…*), die een bevestigingsdialoog opent.
- **Sticky save-balk** onderaan zodra er wijzigingen zijn: links de status (stip in warning plus
  "Unsaved changes"), rechts *Discard* (quiet) en *Save changes* (primair, met een loading-state). Na het opslaan: de toast "Changes saved".

## Mobiele detailpagina (390px)

- Een topbalk met iconknoppen van 44px (terug, favoriet, tas met een teller-badge).
- De media bovenaan, 4:3, randloos, met een chip linksboven ("New") en paginatie-dots.
- Titel, rating (ster in `--star`) en prijs (16/700, tabular-nums), met de voorraadstatus in succes-groen.
- Een keuze met chips (min 48×44). Zonder keuze toont de CTA een foutmelding die zegt wat er moet gebeuren
  ("Choose a size to add this to your bag").
- Detailregels met een icoon van 20px plus tekst (levering, retour).
- **Sticky actiebalk** onderaan: totaal links, de primaire knop rechts (groot, 48px).

## Edge cases (altijd ontwerpen)

| Staat | Hoe |
|---|---|
| Laden | Skeletons in de vorm van de echte content; geen spinner over de hele pagina |
| Leeg | Een kop ("No orders yet"), een zin over wanneer hier iets komt te staan, en een volgende stap als knop |
| Fout | Een kop ("Couldn't load orders"), de oorzaak in gewone taal plus geruststelling ("Your data is safe"), *Try again* (primair) en een tweede uitweg |
| Bezig | Een knop met een spinner en een werkwoord met "…"; voorkom dubbel indrukken |
| Klaar | Een toast met een bevestiging in het verleden ("Changes saved") |
