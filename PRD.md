# PRD: Meta Creatives Pipeline

## Oversikt

Et CLI-basert system for å generere store mengder statiske Meta-annonser (creatives) fra HTML/CSS-templates og strukturert data. Systemet skal brukes internt av Solid Media først, men designes for å enkelt onboarde nye kunder/brands.

**Primært mål:** Gå fra kampanjebriefing til 30-50+ ferdige creatives på minutter, ikke timer.

## Arkitektur

```
meta-creatives/
├── templates/              # HTML/CSS ad-templates
│   ├── awareness/          # Gruppert etter kampanjetype
│   │   ├── bold-statement.html
│   │   ├── social-proof.html
│   │   └── problem-agitate.html
│   ├── conversion/
│   │   ├── offer-cta.html
│   │   └── comparison.html
│   └── _partials/          # Gjenbrukbare HTML-fragmenter (logo, CTA-knapp, badge)
│       ├── cta-button.html
│       └── logo-block.html
├── brands/                 # Merkevare-konfigurasjon
│   ├── solid-media.json
│   └── _example.json
├── campaigns/              # Kampanjedata
│   ├── solid-media-q2-awareness.json
│   └── _example.json
├── output/                 # Genererte creatives (gitignored)
├── src/
│   ├── cli.ts              # CLI entry point
│   ├── renderer.ts         # Puppeteer rendering engine
│   ├── template-engine.ts  # Template injection og variasjonsgenerering
│   ├── matrix.ts           # Kombinatorisk variant-matrise
│   └── naming.ts           # Filnavnkonvensjon
├── package.json
├── tsconfig.json
└── README.md
```

## Tech stack

- **Runtime:** Node.js med TypeScript
- **Rendering:** Puppeteer (headless Chrome → screenshot)
- **Template:** Ren HTML/CSS med Handlebars for variabelinjeksjon
- **CLI:** Commander.js
- **Bildeprosessering:** Sharp (for eventuell optimalisering/resizing)

## Datamodell

### Brand-konfigurasjon (`brands/solid-media.json`)

```json
{
  "name": "Solid Media",
  "slug": "solid-media",
  "colors": {
    "primary": "#1A1A2E",
    "secondary": "#E94560",
    "textLight": "#FFFFFF"
  },
  "fonts": {
    "heading": {
      "family": "Inter",
      "weight": "700",
      "source": "google"
    },
    "body": {
      "family": "Inter",
      "weight": "400",
      "source": "google"
    }
  },
  "logo": {
    "primary": "assets/logos/solid-media-dark.svg",
    "light": "assets/logos/solid-media-light.svg",
    "icon": "assets/logos/solid-media-icon.svg"
  },
  "cta": {
    "style": "rounded",
    "defaultText": "Les mer"
  }
}
```

### Kampanjedata (`campaigns/solid-media-q2-awareness.json`)

```json
{
  "brand": "solid-media",
  "campaign": "q2-awareness",
  "template": "awareness/bold-statement",
  "formats": ["1080x1080", "1080x1920", "1200x628"],
  "variants": {
    "hooks": [
      { "id": "hook-a", "headline": "Din nettside er ikke bygget for AI-søk", "subline": "Er du synlig der kundene faktisk leter?" },
      { "id": "hook-b", "headline": "Google forandret alt. Igjen.", "subline": "Slik tilpasser du deg AI Overviews" },
      { "id": "hook-c", "headline": "8% av klikkene forsvant", "subline": "Vi hjelper deg å få dem tilbake" }
    ],
    "images": [
      { "id": "img-serp", "src": "assets/images/serp-screenshot.png", "alt": "AI Overview eksempel" },
      { "id": "img-dashboard", "src": "assets/images/dashboard.png", "alt": "Rank tracker dashboard" }
    ],
    "colorThemes": [
      { "id": "dark", "overrides": { "background": "#1A1A2E", "text": "#FFFFFF" } },
      { "id": "light", "overrides": { "background": "#FFFFFF", "text": "#1A1A2E" } }
    ]
  }
}
```

### Variant-matrise

Systemet genererer det kartesiske produktet av alle variant-dimensjoner:

```
hooks (3) × images (2) × colorThemes (2) × formats (3) = 36 creatives
```

Brukeren kan også spesifisere eksplisitte kombinasjoner for å begrense output med en valgfri `"include"` eller `"exclude"` array i kampanjefilen.

## Templates

### Template-format

Templates er standard HTML/CSS-filer med Handlebars-variabler. Hver template er self-contained med inline CSS.

```html
<!DOCTYPE html>
<html>
  <style>
    @import url('https://fonts.googleapis.com/css2?family={{fonts.heading.family}}:wght@{{fonts.heading.weight}}&family={{fonts.body.family}}:wght@{{fonts.body.weight}}&display=swap');

    * { margin: 0; padding: 0; box-sizing: border-box; }

    .ad {
      width: {{format.width}}px;
      height: {{format.height}}px;
      background: {{colors.background}};
      color: {{colors.text}};
      font-family: '{{fonts.body.family}}', sans-serif;
      display: flex;
      flex-direction: column;
      justify-content: space-between;
      padding: {{format.padding}}px;
      overflow: hidden;
      position: relative;
    }

    .headline {
      font-family: '{{fonts.heading.family}}', sans-serif;
      font-weight: {{fonts.heading.weight}};
      font-size: {{format.headlineSize}}px;
      line-height: 1.1;
      letter-spacing: -0.02em;
    }

    .subline {
      font-size: {{format.sublineSize}}px;
      opacity: 0.85;
      margin-top: 12px;
    }

    .cta-button {
      display: inline-block;
      background: {{colors.secondary}};
      color: {{colors.textLight}};
      padding: 14px 32px;
      border-radius: {{#if (eq cta.style "rounded")}}8px{{else}}0{{/if}};
      font-weight: 600;
      font-size: {{format.ctaSize}}px;
    }

    .hero-image {
      width: 100%;
      flex: 1;
      object-fit: cover;
      border-radius: 8px;
      margin: 16px 0;
    }

    .logo {
      height: 28px;
      width: auto;
    }
  </style>
</head>
<body>
  <div class="ad">
    <div class="top-bar">
      <img class="logo" src="{{logo.current}}" alt="{{brand.name}}" />
    </div>
    <div class="content">
      <h1 class="headline">{{hook.headline}}</h1>
      <p class="subline">{{hook.subline}}</p>
    </div>
    {{#if image}}
    <img class="hero-image" src="{{image.src}}" alt="{{image.alt}}" />
    {{/if}}
    <div class="bottom-bar">
      <span class="cta-button">{{cta.text}}</span>
    </div>
  </div>
</body>
</html>
```

### Format-responsive verdier

Hvert format har forhåndsdefinerte responsive verdier slik at typografi og spacing skalerer riktig:

```json
{
  "1080x1080": { "width": 1080, "height": 1080, "padding": 60, "headlineSize": 52, "sublineSize": 22, "ctaSize": 18 },
  "1080x1920": { "width": 1080, "height": 1920, "padding": 64, "headlineSize": 56, "sublineSize": 24, "ctaSize": 20 },
  "1200x628":  { "width": 1200, "height": 628,  "padding": 48, "headlineSize": 42, "sublineSize": 18, "ctaSize": 16 }
}
```

## CLI-grensesnitt

### Kommandoer

```bash
# Generer alle varianter for en kampanje
npx meta-creatives generate campaigns/solid-media-q2-awareness.json

# Generer kun ett format
npx meta-creatives generate campaigns/solid-media-q2-awareness.json --format 1080x1080

# Generer med spesifikke varianter
npx meta-creatives generate campaigns/solid-media-q2-awareness.json --hook hook-a --theme dark

# Preview én spesifikk kombinasjon i nettleseren (åpner HTML)
npx meta-creatives preview campaigns/solid-media-q2-awareness.json --hook hook-a --image img-serp --theme dark --format 1080x1080

# Liste ut variant-matrisen uten å rendre
npx meta-creatives matrix campaigns/solid-media-q2-awareness.json

# Scaffold nytt brand
npx meta-creatives init-brand "Kundenavn"

# Scaffold ny kampanje
npx meta-creatives init-campaign solid-media "kampanjenavn"
```

### Output

```
output/
└── solid-media/
    └── q2-awareness/
        ├── solid-media_q2-awareness_bold-statement_1080x1080_hook-a_img-serp_dark.png
        ├── solid-media_q2-awareness_bold-statement_1080x1080_hook-a_img-serp_light.png
        ├── solid-media_q2-awareness_bold-statement_1080x1080_hook-a_img-dashboard_dark.png
        ├── ...
        └── manifest.json   # Metadata om alle genererte filer
```

### Filnavnkonvensjon

```
{brand}_{campaign}_{template}_{format}_{hook}_{image}_{theme}.png
```

Alt lowercase, bindestreker innenfor segmenter, understreker mellom segmenter. Gjør det trivielt å filtrere og sortere i Meta Ads Manager.

### manifest.json

```json
{
  "generated": "2026-03-22T14:30:00Z",
  "brand": "solid-media",
  "campaign": "q2-awareness",
  "totalVariants": 36,
  "files": [
    {
      "filename": "solid-media_q2-awareness_bold-statement_1080x1080_hook-a_img-serp_dark.png",
      "format": "1080x1080",
      "hook": "hook-a",
      "image": "img-serp",
      "theme": "dark",
      "headline": "Din nettside er ikke bygget for AI-søk",
      "dimensions": { "width": 1080, "height": 1080 },
      "sizeKb": 245
    }
  ]
}
```

## Renderingsmotor

### Flyt

```
1. Les kampanje-JSON
2. Les brand-JSON (basert på kampanjens `brand`-felt)
3. Merge brand-farger med eventuelle colorTheme-overrides
4. Bygg variant-matrise (kartesisk produkt)
5. For hver variant:
   a. Les template HTML
   b. Kompiler Handlebars-template med merged data (brand + variant + format)
   c. Start Puppeteer-side med riktig viewport (format-dimensjoner)
   d. Sett HTML-innhold
   e. Vent på fonter og bilder (networkidle0)
   f. Ta screenshot som PNG
   g. Optimaliser med Sharp (komprimering uten kvalitetstap)
   h. Lagre til output-mappe
6. Generer manifest.json
7. Skriv oppsummering til terminal
```

### Ytelse

- Gjenbruk én Puppeteer browser-instans på tvers av alle varianter
- Bruk `page.setContent()` og viewport-resize fremfor ny side per variant
- Parallelliser med et konfigurerbart antall concurrent sider (default: 4)
- Mål og rapporter total render-tid og gjennomsnitt per creative

## Håndtering av assets

- Bilder referert i kampanje-JSON bruker relative paths fra prosjektrot
- Logo-paths i brand-JSON er også relative fra prosjektrot
- Alle asset-paths konverteres til `file://`-absolutte paths ved rendering
- Google Fonts lastes via `@import` i template CSS

## Validering

Før rendering, valider:

- At brand-JSON eksisterer og har alle required fields
- At kampanje-JSON er gyldig og refererer til eksisterende brand
- At alle refererte template-filer eksisterer
- At alle refererte asset-filer (bilder, logoer) eksisterer
- At format-dimensjoner er gyldige positive heltall
- Gi klare feilmeldinger med filsti og felt som mangler

## Fremtidige utvidelser (ikke i scope for v1, men design for det)

- **Claude API-integrasjon for copy-generering:** Legg til en `generate-copy` kommando som tar et kampanjebrief og genererer hook-varianter via Claude API, lagrer direkte som kampanje-JSON.
- **Figma-import:** Les farger og fonter fra en Figma-fil via API for å auto-generere brand-JSON.
- **Video/motion:** Utvide til å generere korte animerte varianter med Remotion eller liknende.
- **A/B-test tracking:** Koble manifest.json med Meta API for å automatisk opprette ad sets med riktig navngivning.
- **Web UI:** Et enkelt dashboard for å browse og godkjenne genererte creatives før de pushes til Meta.

## Akseptansekriterier for v1

1. `generate`-kommandoen produserer PNG-filer i riktige dimensjoner for alle tre formater (1080x1080, 1080x1920, 1200x628)
2. Fonter fra Google Fonts rendres korrekt i output
3. Bilder (PNG/JPG/SVG) vises korrekt i templates
4. Variant-matrisen genererer korrekt antall kombinasjoner
5. Filnavn følger konvensjonen konsekvent
6. manifest.json inneholder korrekt metadata for alle genererte filer
7. `preview`-kommandoen åpner en spesifikk variant i nettleseren
8. `matrix`-kommandoen viser alle planlagte varianter i terminalen
9. `init-brand` og `init-campaign` scaffolder gyldige JSON-filer
10. Rendering av 36 varianter tar under 60 sekunder
11. Klare feilmeldinger ved ugyldig input (manglende filer, feil JSON-format)
12. README med installasjon, brukseksempler, og forklaring av datamodellen
13. Minst to ferdige templates medfølger: `awareness/bold-statement` og `conversion/offer-cta`
