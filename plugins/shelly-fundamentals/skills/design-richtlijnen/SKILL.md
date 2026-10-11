---
name: design-richtlijnen
description: Visuele huisstijl en UI-regels van shelly-nas apps (het Web App Template): kleuren, Figtree-typografie, spacing, layout, componenten en hun states, dark mode, toegankelijkheid en schermpatronen. Gebruik bij het bouwen of stylen van UI, componenten, pagina's, formulieren, tabellen, dashboards of een thema, of bij het kiezen van kleuren, lettertypes of spacing.
---

# Designrichtlijnen

Bron: het **Web App Template** uit Claude Design (`reference/web-app-template.html`; open het in
een browser voor alle borden). Elke nieuwe app volgt dit, zodat alle apps op de NAS als één familie aanvoelen.

## Bestanden in deze skillmap

| Bestand | Gebruik |
|---|---|
| `tokens.css` | Alle tokens als CSS-variabelen (kleur, spacing, radius, schaduw) voor light en dark. **De bron van waarheid.** |
| `theme.ts` | Dezelfde waarden als MUI-thema (`themeSettings(mode)`), inclusief component-overrides. Zet hem in `client/src/theme.ts`. |
| `components.css` | Referentie-implementatie van alle componenten en states. Dit is de spec voor maten en kleuren; in een MUI-app regelt `theme.ts` dit. |
| `patterns.md` | Schermpatronen: app-shell (desktop en mobiel), lijst/dashboard, instellingenformulier, mobiele detailpagina, edge cases. **Lees dit bij het bouwen van een nieuw scherm.** |

### Zonder MUI (gewone CSS)

Een app die (nog) geen MUI gebruikt, volgt dezelfde regels met `tokens.css` en `components.css`:

- Kopieer `tokens.css` naar `client/src/tokens.css` en importeer in `main.jsx` in deze volgorde:
  `@fontsource/figtree/{400,500,600,700}.css`, `tokens.css` en dan de eigen stylesheet.
  `tokens.css` zet ook de basis van `body` (lettertype, kleur, achtergrond).
- De eigen stylesheet leest **alleen rollen** (`var(--primary)`, `var(--surface)`, `var(--text-muted)`).
  Neem maten, states en kleuren over uit `components.css`; de klassennamen van de app mogen blijven.
- Een bestaande app omzetten: vervang eerst de eigen kleurvariabelen door de rollen (ook waar JSX ze
  als string doorgeeft, bijvoorbeeld `stroke={`var(${kleur})`}`). Zet daarna dark mode van
  `@media (prefers-color-scheme)` om naar `.theme-dark`, anders wint de mediaquery van de toggle.
- **Grafiekkleuren zijn geen UI-rollen.** Houd een gevalideerd reekspalet (`--reeks-1..n`) apart, met
  een eigen waarde onder `.theme-dark`. Een reeks betekent geen actie of status, dus geen `--primary`.

## Principes

1. **Kleur heeft een taak, nooit decoratie.** Eén blauwe primaire kleur voor acties, links en focus.
   Een koele grijsramp voor al het andere. Vier semantische kleuren alleen voor betekenis.
2. **Gebruik rollen, geen hexwaarden.** Componenten lezen `--primary`, `--surface`, `--text-muted`
   (of `theme.palette`). Een andere theme verandert de waarden, nooit de markup.
3. **Eén accent.** Als alles blauw is, is niets klikbaar.
4. **Status nooit alleen met kleur.** Altijd een stip of icoon plus een woord.
5. **Witruimte groepeert, geen lijnen.** Strak binnen een groep, ruim tussen groepen.
6. **Elke control laat zien wat hij doet, en reageert:** hover, focus, pressed, loading en disabled.
7. **Ontwerp de momenten ertussen:** loading, leeg en fout vertellen wat er gebeurt en wat de gebruiker kan doen.

## Kleur

- **Primair (blauw):** `600 #1F4FC4` = actie, `700` = hover, `800` = pressed, `500 #2F64E0` = focusring,
  `50`/`100` = getinte achtergronden en focushalo. Een andere merkkleur? Vervang de ramp door een tint op dezelfde lichtheidsstappen; de semantische kleuren blijven.
- **Neutraal:** achtergrond `#F8F9FB`, oppervlak `#FFFFFF`, koppen `#12161C`. Bodytekst gebruikt
  grey-700 `#343B48`, niet puur zwart. Muted tekst `#5F6878` haalt het contrast op elk licht oppervlak.
- **Semantisch**, elk met één betekenis, altijd als paar met een zachte achtergrond:
  - Blauw: info, focus, selectie
  - Groen `#1A7F4B`: succes, bevestiging, gezonde status
  - Geel: waarschuwing. De vulling is `#F5B83D`, de **tekst** donker amber `#9A5200` vanwege het contrast.
  - Rood `#C8322B`: fouten, destructieve acties en blokkerende problemen. Niets anders.
- **Contrast:** tekst minimaal 4.5:1, grote tekst en iconen 3:1.

## Dark mode

- **Verplicht.** Volg standaard `prefers-color-scheme` en laat de gebruiker wisselen. Zet de klasse
  `theme-dark` of `theme-light` op `<html>` (of geef de modus door aan `themeSettings(mode)`) en onthoud de keuze in `localStorage`.
- **Zet de klasse vóór de eerste paint**, met een klein inline `<script>` in de `<head>` van `index.html`
  (eerst `localStorage`, anders `matchMedia('(prefers-color-scheme: dark)')`). Pas het in React zetten
  geeft een witte flits bij elke laadbeurt in dark mode. Zonder eigen keuze volgt de app een
  systeemwissel live (`matchMedia(...).addEventListener('change', ...)`); een klik op de toggle legt de
  keuze vast. Lees en schrijf `localStorage` altijd in `try/catch`.
- Diepte komt van **lichtere oppervlakken**, niet van schaduwen: bg `#0F1217` → kaart `#171B22` →
  menu `#1F242D` → popover/toast `#282E39`. In dark mode zijn er geen schaduwen.
- Accenten zijn gedesatureerd en lichter (primair `#7EA2F0`), om verblinding te voorkomen.

## Typografie

Eén familie, **Figtree** (400/500/600/700), self-hosted via `@fontsource/figtree`.
Zes groottes, niet meer:

| Stijl | Grootte / gewicht | Regelhoogte | Tracking | Gebruik | MUI |
|---|---|---|---|---|---|
| Display | 32 / Bold | 115% | −2.5% | Eén per pagina; lege staten | `h1` |
| Title | 24 / Semibold | 120% | −2% | Sectie- of dashboardkop; grootste in dichte dashboards | `h2` |
| Heading | 20 / Semibold | 125% | −1.5% | Kaarten, dialogen, formuliersecties | `h3` |
| Body | 16 / Regular | 150% | 0 | Alinea's en inputwaarden, in grey-700 | `body1` |
| Small | 14 / Reg of Semi | 150% | 0 | Knoppen, tabelcellen, secundaire tekst, metadata | `body2` |
| Label | 12 / Semibold, HOOFDLETTERS | 135% | +6% | Overlines en kolomkoppen. **Nooit voor zinnen.** | `overline` |

- Getallen in tabellen en prijzen: `font-variant-numeric: tabular-nums`, rechts uitgelijnd.
- Teksten in **sentence case**, ook op knoppen. Nooit alles in hoofdletters, behalve bij Label.

## Spacing, layout, radius en elevatie

- **Alles is een veelvoud van 4.** Schaal: 4 (icoon↔label), 8 (titel↔subtekst), 12 (compacte lijsten),
  16 (items in een groep), 24 (padding van kaarten, gutters), 32 (tussen groepen), 48 (tussen paginasecties), 64 (bovenkant pagina).
  In MUI is `spacing: 4`, dus `sx={{ p: 6 }}` = 24px.
- **Grid:** 12 kolommen met gutters van 24px. Zijmarge 24px op telefoons, 32px in het content-gebied van de app.
- **Radius:** 6 voor kleine knoppen en chips, 8 voor knoppen, inputs, nav-items en alerts, 12 voor kaarten,
  dialogen en toasts, volledig rond voor badges, toggles en avatars.
- **Elevatie (light):** 0 = alleen een rand; 1 = kaarten (`--shadow-1`); 2 = menu's, toasts en kaarten bij hover (`--shadow-2`).

## Componenten

**Knoppen.** Eén primaire actie per view. Secundair is outlined, tertiair is een quiet tekstknop,
destructief is outlined in rood. Hoogtes: 32 (klein), 40 (standaard), 48 (groot). Tijdens loading
krijgt de knop een spinner en een werkwoord met "…" (*Saving…*), en kan hij niet nog een keer ingedrukt worden.
Pressed schuift 1px omlaag. Focus is een dubbele ring (2px oppervlak, 2px `--focus`).

**Invoervelden.** Het label staat erboven (14/600), help- of fouttekst eronder (12px). Hoogte 40,
tekstgrootte 16 (dat voorkomt zoomen op iOS). Bij hover wordt de rand donkerder, focus geeft een
blauwe rand en een halo van 3px. Een fout geeft een rode rand, een icoon en een boodschap die zegt **hoe je het oplost**
("Enter a full email, like jane@company.com"). Een disabled veld legt uit waarom het op slot zit.

**Selectie.** Een geselecteerde optie heeft altijd **twee signalen**: kleur plus een onderstreping,
vulling, ring of vinkje. Tabs hebben een onderstreping van 2px plus een vet label, met optioneel een teller.
Een chip die niet beschikbaar is, wordt doorgestreept en is niet klikbaar.

**Navigatie.** Het huidige item krijgt een getinte achtergrond (`--primary-soft`) en een vet label.
Een disabled item kan het label "Soon" krijgen.

**Status.** Badges voor compacte status (stip plus woord: Paid, Pending, Failed, New, Draft);
alerts voor boodschappen (icoon plus zin, zachte achtergrond).

**Tabellen.** Kolomkoppen in de Label-stijl, cellen 14px met padding 12×24, rijen met hover-tint,
numerieke kolommen rechts uitgelijnd met tabular-nums. Zet de tabel in een kaart die horizontaal kan scrollen.

**Toasts.** Onderaan gecentreerd, donker, radius 12, met een succesicoon. Na ongeveer 2,4 seconden verdwijnen ze.
Gebruik ze voor bevestigingen van acties ("Changes saved", "Link copied").

**Tekst op afbeeldingen.** Tekst staat op een gradient onderaan; iconen en tags staan in effen
chips (wit met 94% dekking), zodat ze op elke foto leesbaar zijn.

## Toegankelijkheid

- Tap-targets minimaal **44px** (tabs, chips, switches en checkboxen hebben een min-height van 44).
- Zichtbare focus op alles wat interactief is, met `:focus-visible`. Haal de outline nooit weg zonder een vervanging.
- Status nooit alleen met kleur; iconen krijgen `aria-hidden`, en betekenisvolle iconknoppen een `aria-label`.
- Contrast volgens de regels hierboven. Controleer nieuwe kleurcombinaties.

## Animatie

Overgangen van 150 ms `ease` op kleur, rand en schaduw. Toasts en pop-ins gebruiken
`cubic-bezier(.2,.8,.2,1)` in 240 ms. Skeletons glanzen in 1,4 s. Geen animatie die alleen versiering is.
