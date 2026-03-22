# PRD: Meta Creatives Pipeline

## Overview

A CLI-based system for generating large volumes of static Meta ad creatives from HTML/CSS templates and structured data. The system will be used internally by Solid Media first, but is designed to easily onboard new clients/brands.

**Primary goal:** Go from campaign briefing to 30-50+ finished creatives in minutes, not hours.

## Architecture

```
meta-creatives/
├── templates/              # HTML/CSS ad templates
│   ├── awareness/          # Grouped by campaign type
│   │   ├── bold-statement.html
│   │   ├── social-proof.html
│   │   └── problem-agitate.html
│   ├── conversion/
│   │   ├── offer-cta.html
│   │   └── comparison.html
│   └── _partials/          # Reusable HTML fragments (logo, CTA button, badge)
│       ├── cta-button.html
│       └── logo-block.html
├── brands/                 # Brand configuration
│   ├── solid-media.json
│   └── _example.json
├── campaigns/              # Campaign data
│   ├── solid-media-q2-awareness.json
│   └── _example.json
├── output/                 # Generated creatives (gitignored)
├── src/
│   ├── cli.ts              # CLI entry point
│   ├── renderer.ts         # Puppeteer rendering engine
│   ├── template-engine.ts  # Template injection and variation generation
│   ├── matrix.ts           # Combinatorial variant matrix
│   └── naming.ts           # File naming convention
├── package.json
├── tsconfig.json
└── README.md
```

## Tech Stack

- **Runtime:** Node.js with TypeScript
- **Rendering:** Puppeteer (headless Chrome → screenshot)
- **Templating:** Plain HTML/CSS with Handlebars for variable injection
- **CLI:** Commander.js
- **Image processing:** Sharp (for optional optimization/resizing)

## Data Model

### Brand Configuration (`brands/solid-media.json`)

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
    "defaultText": "Learn more"
  }
}
```

### Campaign Data (`campaigns/solid-media-q2-awareness.json`)

```json
{
  "brand": "solid-media",
  "campaign": "q2-awareness",
  "template": "awareness/bold-statement",
  "formats": ["1080x1080", "1080x1920", "1200x628"],
  "variants": {
    "hooks": [
      { "id": "hook-a", "headline": "Your website isn't built for AI search", "subline": "Are you visible where customers actually look?" },
      { "id": "hook-b", "headline": "Google changed everything. Again.", "subline": "How to adapt to AI Overviews" },
      { "id": "hook-c", "headline": "8% of clicks disappeared", "subline": "We help you get them back" }
    ],
    "images": [
      { "id": "img-serp", "src": "assets/images/serp-screenshot.png", "alt": "AI Overview example" },
      { "id": "img-dashboard", "src": "assets/images/dashboard.png", "alt": "Rank tracker dashboard" }
    ],
    "colorThemes": [
      { "id": "dark", "overrides": { "background": "#1A1A2E", "text": "#FFFFFF" } },
      { "id": "light", "overrides": { "background": "#FFFFFF", "text": "#1A1A2E" } }
    ]
  }
}
```

### Variant Matrix

The system generates the Cartesian product of all variant dimensions:

```
hooks (3) × images (2) × colorThemes (2) × formats (3) = 36 creatives
```

Users can also specify explicit combinations to limit output with an optional `"include"` or `"exclude"` array in the campaign file.

## Templates

### Template Format

Templates are standard HTML/CSS files with Handlebars variables. Each template is self-contained with inline CSS.

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

### Format-Responsive Values

Each format has predefined responsive values so that typography and spacing scale correctly:

```json
{
  "1080x1080": { "width": 1080, "height": 1080, "padding": 60, "headlineSize": 52, "sublineSize": 22, "ctaSize": 18 },
  "1080x1920": { "width": 1080, "height": 1920, "padding": 64, "headlineSize": 56, "sublineSize": 24, "ctaSize": 20 },
  "1200x628":  { "width": 1200, "height": 628,  "padding": 48, "headlineSize": 42, "sublineSize": 18, "ctaSize": 16 }
}
```

## CLI Interface

### Commands

```bash
# Generate all variants for a campaign
npx meta-creatives generate campaigns/solid-media-q2-awareness.json

# Generate only one format
npx meta-creatives generate campaigns/solid-media-q2-awareness.json --format 1080x1080

# Generate with specific variants
npx meta-creatives generate campaigns/solid-media-q2-awareness.json --hook hook-a --theme dark

# Preview a specific combination in the browser (opens HTML)
npx meta-creatives preview campaigns/solid-media-q2-awareness.json --hook hook-a --image img-serp --theme dark --format 1080x1080

# List the variant matrix without rendering
npx meta-creatives matrix campaigns/solid-media-q2-awareness.json

# Scaffold a new brand
npx meta-creatives init-brand "Client Name"

# Scaffold a new campaign
npx meta-creatives init-campaign solid-media "campaign-name"
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
        └── manifest.json   # Metadata for all generated files
```

### File Naming Convention

```
{brand}_{campaign}_{template}_{format}_{hook}_{image}_{theme}.png
```

All lowercase, hyphens within segments, underscores between segments. Makes it trivial to filter and sort in Meta Ads Manager.

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
      "headline": "Your website isn't built for AI search",
      "dimensions": { "width": 1080, "height": 1080 },
      "sizeKb": 245
    }
  ]
}
```

## Rendering Engine

### Flow

```
1. Read campaign JSON
2. Read brand JSON (based on the campaign's `brand` field)
3. Merge brand colors with any colorTheme overrides
4. Build variant matrix (Cartesian product)
5. For each variant:
   a. Read template HTML
   b. Compile Handlebars template with merged data (brand + variant + format)
   c. Launch Puppeteer page with correct viewport (format dimensions)
   d. Set HTML content
   e. Wait for fonts and images (networkidle0)
   f. Take screenshot as PNG
   g. Optimize with Sharp (lossless compression)
   h. Save to output directory
6. Generate manifest.json
7. Write summary to terminal
```

### Performance

- Reuse a single Puppeteer browser instance across all variants
- Use `page.setContent()` and viewport resize instead of a new page per variant
- Parallelize with a configurable number of concurrent pages (default: 4)
- Measure and report total render time and average per creative

## Asset Handling

- Images referenced in campaign JSON use relative paths from project root
- Logo paths in brand JSON are also relative from project root
- All asset paths are converted to `file://` absolute paths at render time
- Google Fonts are loaded via `@import` in template CSS

## Validation

Before rendering, validate:

- That the brand JSON exists and has all required fields
- That the campaign JSON is valid and references an existing brand
- That all referenced template files exist
- That all referenced asset files (images, logos) exist
- That format dimensions are valid positive integers
- Provide clear error messages with file path and missing field

## Future Extensions (not in scope for v1, but design for it)

- **Claude API integration for copy generation:** Add a `generate-copy` command that takes a campaign brief and generates hook variants via Claude API, saving directly as campaign JSON.
- **Figma import:** Read colors and fonts from a Figma file via API to auto-generate brand JSON.
- **Video/motion:** Extend to generate short animated variants with Remotion or similar.
- **A/B test tracking:** Connect manifest.json with Meta API to automatically create ad sets with correct naming.
- **Web UI:** A simple dashboard to browse and approve generated creatives before pushing to Meta.

## Acceptance Criteria for v1

1. The `generate` command produces PNG files in correct dimensions for all three formats (1080x1080, 1080x1920, 1200x628)
2. Fonts from Google Fonts render correctly in output
3. Images (PNG/JPG/SVG) display correctly in templates
4. The variant matrix generates the correct number of combinations
5. File names follow the convention consistently
6. manifest.json contains correct metadata for all generated files
7. The `preview` command opens a specific variant in the browser
8. The `matrix` command displays all planned variants in the terminal
9. `init-brand` and `init-campaign` scaffold valid JSON files
10. Rendering 36 variants takes under 60 seconds
11. Clear error messages for invalid input (missing files, bad JSON format)
12. README with installation, usage examples, and data model explanation
13. At least two ready-made templates included: `awareness/bold-statement` and `conversion/offer-cta`
