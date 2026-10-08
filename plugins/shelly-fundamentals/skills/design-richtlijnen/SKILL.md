---
name: design-richtlijnen
description: Visuele huisstijl van shelly-nas apps: kleuren, typografie, MUI-thema en light/dark mode. Gebruik bij het bouwen of stylen van UI, componenten, pagina's, grafieken of een thema, of bij het kiezen van kleuren en lettertypes.
---

# Designrichtlijnen

Afgeleid uit FinanceApp. Nieuwe apps nemen dit over, zodat alle apps op de NAS als één familie aanvoelen.

## Bestanden in deze skillmap

- `expanded-theme.ts`: TypeScript-uitbreidingen voor `palette.cosmetics` en de typografievarianten `body3`, `credit` en `debit`. Zet hem in `client/src/`.
- `theme.ts`: het MUI-thema (`themeSettings(mode)`) met de tokens. Zet hem in `client/src/theme.ts`.
- `tokens.css`: dezelfde tokens als CSS custom properties, voor plekken buiten MUI.

## Kern

- **Lettertype:** *Reddit Mono* (Google Fonts) voor alles, met `monospace` als fallback. Laad het in `index.html`.
- **Basisgrootte:** 12px. Koppen h2/h3 zijn 13px, vet en in HOOFDLETTERS. h1 is 32px en vet.
- **Primair:** mintgroen, ramp `primary.100`–`900`. `main` = `600` in light mode, `500` in dark mode.
- **Secundair:** geel/oker, ramp `secondary.100`–`900`. `main` = `500`.
- **Neutraal:** grijsramp `grey.100`–`900`. In dark mode wordt de ramp omgedraaid, zodat lage
  sleutels dicht bij de tekstkleur blijven en hoge sleutels dicht bij het oppervlak.
- **Achtergrond:** light `#fbfbfb` met papier `#ffffff`; dark `#15171b` met papier `#1e2126`.
- **Positief/negatief bedrag:** positief = primair (`600` light, `400` dark); negatief = `#E35335` light, `#ff7a5e` dark.
- **Vorm:** border-radius 5px, randen van 1px solid in `grey.200` (light) of `grey.700` (dark).

## Regels

- **Altijd light én dark mode.** De modus volgt de systeemvoorkeur en is te wisselen via een
  `ColorModeContext`. Hardcode geen kleuren in componenten; gebruik `theme.palette`.
- Gebruik MUI-componenten in plaats van eigen varianten van knoppen, inputs of tabellen.
  Voor tabellen met veel data: `@mui/x-data-grid`.
- Grafieken: één library per app (Recharts heeft de voorkeur), met kleuren uit de tokens.
- Dashboards zijn compact en informatiedicht. Geen grote hero-blokken of decoratieve illustraties.
- Elke pagina moet bruikbaar zijn op telefoonbreedte.
