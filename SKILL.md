---
name: ui-template-library
description: >
  Generate a production-ready HTML/CSS website or app UI from a curated library of real-world design systems. Trigger when the user wants to build, design, or generate a webpage, landing page, dashboard, app screen, or any UI — especially for industries like finance, banking, healthcare, insurance, e-commerce, or SaaS. Automatically picks the closest reference template, loads its design tokens and screenshot, applies any user style overrides (font, color, etc.), and outputs a complete single-file HTML page styled to look like a real product. Includes Chart.js charts styled to match the template for data-heavy pages like dashboards and finance apps.
---

# UI Design Library Skill

## Purpose
Generate production-ready HTML/CSS pages styled from a curated reference library of real-world design systems. The output should look like a professional product, not a generic template.

---

## Step 1 — Understand the Request

Read the user's requirement carefully. Extract:
- **What** they want to build (landing page, dashboard, app screen, form, etc.)
- **The industry or domain** (finance, healthcare, e-commerce, SaaS, creative, etc.)
- **Any explicit style overrides** the user mentioned (e.g. a specific font, a colour, a radius style)

Keep a mental note of overrides — they take precedence over the template later.

---

## Step 2 — Pick the Right Template

### Available industry → template mapping

Scan the `references/` directory structure:

```
references/
  Finance/
    antbank/    antbank.json  antbank.png   → Corporate digital bank, vivid blue, clean flat UI
    futunn/     futunn.json   futunn.png    → Stock trading app, data-dense, dark accents
    mox/        mox.json      mox.png       → Neo-bank, neon cyan, editorial, youthful
    visa/       visa.json     visa.png      → Global payments brand, authoritative blue/gold
  Healthcare/
    bowtie/     bowtie.json   bowtie.png    → Insurtech, bold pink/purple gradient, modern
  Creative/     (reserved — no templates yet)
  E-commerce/   (reserved — no templates yet)
  Lifestyle/    (reserved — no templates yet)
  Services/     (reserved — no templates yet)
  Tech & SaaS/  (reserved — no templates yet)
```

### Selection logic

| User request signals | Best match |
|---|---|
| Banking, payments, finance app, fintech (youthful / neo-bank) | `Finance/mox` |
| Banking, finance (corporate / institutional) | `Finance/antbank` |
| Stock trading, investment, brokerage, market data | `Finance/futunn` |
| Payments, card network, enterprise finance | `Finance/visa` |
| Insurance, health insurance, insurtech | `Healthcare/bowtie` |
| Healthcare, clinic, medical, wellness | `Healthcare/bowtie` |
| Any industry with no matching template | Pick the closest available; note the substitution to the user |

Once selected, do **both** of the following:
1. Read the `.json` file with the Read tool
2. Read the `.png` screenshot image with the Read tool (view it visually to absorb the aesthetic)

---

## Step 3 — Apply Style Overrides

After loading the template tokens, apply any user-specified overrides:

- **Font family**: replace `typography.fontFamilies.heading` and `.body` with the requested font. Add the appropriate Google Fonts `<link>` tag. Keep all other type tokens (sizes, weights, line-heights) from the template unless the user also specified them.
- **Primary colour**: replace `color.brand.primary` and any derived tokens (button backgrounds, borders, focus states, active nav colours) with the user's colour. Derive a hover shade (~15% darker) algorithmically.
- **Border radius style**: if the user says "sharp" set all radii to ≤4px; "rounded" means cards ~16px, buttons ~8px; "pill" means buttons ~9999px.
- **Dark / light mode**: if the user specifies a mode, adjust background and text tokens accordingly.
- Anything else the user specifies overrides the matching token; everything else comes from the template.

Do **not** ask for confirmation — apply the override and proceed.

---

## Step 4 — Generate the HTML/CSS

### Constraints
- Single self-contained `.html` file (inline `<style>` block, no external CSS frameworks).
- Use CSS custom properties (`--token-name`) mapped directly from the JSON tokens.
- Use Google Fonts `<link>` for any web-safe font substitution (the template's custom/proprietary fonts won't load locally).
- Responsive: mobile-first with at least one breakpoint at ~768px.
- No JavaScript required for layout; JS only if the user explicitly asks for interactions.

### Structure to follow

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title><!-- page title --></title>
  <!-- Google Fonts link for the chosen font -->
  <!-- Chart.js — include whenever the page has charts or graphs -->
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    /* 1. CSS custom properties (design tokens) */
    :root { ... }

    /* 2. Reset + base */
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: var(--font-body); ... }

    /* 3. Layout utilities */

    /* 4. Component styles (navbar, hero, cards, buttons, inputs, footer …) */

    /* 5. Responsive breakpoints */
  </style>
</head>
<body>
  <!-- Semantic HTML matching the user's requested page/screen -->

  <!-- Chart.js initialisation — reads CSS custom properties so colours stay in sync with the template -->
  <script>
    /* chart setup goes here */
  </script>
</body>
</html>
```

### CSS token mapping

Map JSON fields → CSS custom properties like this:

```css
:root {
  /* Colors */
  --color-primary:        <color.brand.primary>;
  --color-secondary:      <color.brand.secondary>;
  --color-bg:             <color.background.page>;
  --color-surface:        <color.background.card>;
  --color-text:           <color.text.primary>;
  --color-text-secondary: <color.text.secondary>;
  --color-border:         <color.border.default>;
  --color-focus:          <color.border.focus>;

  /* Typography */
  --font-heading: <typography.fontFamilies.heading>;  /* swap proprietary → web font */
  --font-body:    <typography.fontFamilies.body>;
  --text-xs:      <typography.fontSizes.xs>;
  --text-sm:      <typography.fontSizes.sm>;
  --text-base:    <typography.fontSizes.base>;
  --text-lg:      <typography.fontSizes.lg>;
  --text-xl:      <typography.fontSizes.xl>;
  --text-2xl:     <typography.fontSizes.2xl>;
  --text-3xl:     <typography.fontSizes.3xl>;

  /* Spacing */
  --space-xs:  <spacing.scale.xs or .1>;
  --space-sm:  <spacing.scale.sm or .2>;
  --space-md:  <spacing.scale.md or .4>;
  --space-lg:  <spacing.scale.lg or .6>;
  --space-xl:  <spacing.scale.xl or .8>;
  --space-2xl: <spacing.scale.2xl or .12>;

  /* Border radius */
  --radius-sm:   <border.radius.sm>;
  --radius-base: <border.radius.base>;
  --radius-lg:   <border.radius.lg>;
  --radius-pill: <border.radius.pill>;

  /* Shadows */
  --shadow-sm: <shadows.sm>;
  --shadow-md: <shadows.md>;
  --shadow-lg: <shadows.lg>;

  /* Motion */
  --transition: <motion.normal>;
}
```

### Charts and graphs

When the page type or industry implies data visualisation — dashboards, finance apps, health trackers, analytics screens, trading interfaces, admin panels, etc. — include Chart.js charts.

**When to use charts (non-exhaustive)**
- Finance: portfolio performance, spending breakdown, price history, balance trend
- Healthcare: vitals over time, appointment frequency, health score gauge
- Dashboards / analytics: KPI trends, category breakdowns, usage over time
- Any page with stats, metrics, or time-series data

**Loading Chart.js**

Add this `<script>` tag in `<head>` (CDN, no download needed):
```html
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
```

**Theming rules — charts must match the template**

Extract the following from the loaded JSON and apply them to every chart:

| Chart property | Token to use |
|---|---|
| Dataset primary color (fill / bar / line) | `color.brand.primary` |
| Dataset secondary color | `color.brand.secondary` or `color.brand.accent` |
| Gradient fills | Use `color.gradient.primary` if defined, else build a canvas gradient from `primary` → transparent |
| Grid lines | `color.border.default` at ~30% opacity |
| Tick / label text | `color.text.secondary` |
| Tooltip background | `color.background.card` |
| Tooltip text | `color.text.primary` |
| Tooltip border | `color.border.default` |
| Font family | Match `typography.fontFamilies.body` (use the Google Fonts substitute) |
| Font size | `typography.fontSizes.sm` for ticks, `typography.fontSizes.base` for tooltips |

**Implementation pattern**

```html
<canvas id="myChart"></canvas>

<script>
  // Pull token values from CSS custom properties so chart colours stay in sync
  const style = getComputedStyle(document.documentElement);
  const primary   = style.getPropertyValue('--color-primary').trim();
  const secondary = style.getPropertyValue('--color-secondary').trim();
  const textMuted = style.getPropertyValue('--color-text-secondary').trim();
  const border    = style.getPropertyValue('--color-border').trim();
  const surface   = style.getPropertyValue('--color-surface').trim();

  // Optional: gradient fill for line charts
  const ctx = document.getElementById('myChart').getContext('2d');
  const gradient = ctx.createLinearGradient(0, 0, 0, 300);
  gradient.addColorStop(0, primary + '55');   // ~33% opacity
  gradient.addColorStop(1, primary + '00');   // transparent

  new Chart(ctx, {
    type: 'line',  // or 'bar', 'doughnut', 'radar', etc.
    data: {
      labels: [...],
      datasets: [{
        label: 'Dataset',
        data: [...],
        borderColor: primary,
        backgroundColor: gradient,
        borderWidth: 2,
        pointBackgroundColor: primary,
        tension: 0.4
      }]
    },
    options: {
      responsive: true,
      plugins: {
        legend: {
          labels: { color: textMuted, font: { family: 'var(--font-body)' } }
        },
        tooltip: {
          backgroundColor: surface,
          titleColor: style.getPropertyValue('--color-text').trim(),
          bodyColor: textMuted,
          borderColor: border,
          borderWidth: 1
        }
      },
      scales: {
        x: {
          ticks: { color: textMuted },
          grid:  { color: border + '4D' }   // border at ~30% opacity
        },
        y: {
          ticks: { color: textMuted },
          grid:  { color: border + '4D' }
        }
      }
    }
  });
</script>
```

**Chart selection by context**

| Data type | Recommended chart |
|---|---|
| Trend over time (price, balance, vitals) | `line` with gradient fill |
| Category breakdown (spending, allocation) | `doughnut` or `bar` |
| Comparison across groups | `bar` (horizontal or vertical) |
| Single KPI vs. target | `doughnut` with cutout or gauge pattern |
| Multi-metric radar | `radar` |

Use realistic sample data that fits the industry. Keep dataset arrays at 7–12 data points for readability.

### Content guidelines

- Use realistic placeholder content that fits the industry (not "Lorem ipsum").
- Include all standard sections the user requested. If they only said "landing page", default to: Navbar → Hero → Feature/Benefits section → Social proof (stats or testimonials) → CTA → Footer.
- Match the visual density of the reference screenshot: if the screenshot is data-dense (futunn), be dense; if it's airy and editorial (mox), use generous whitespace.
- Use SVG inline icons or Unicode symbols rather than icon font dependencies.
- Images: use `background-color` blocks with a label or a placeholder `<div>` with the correct aspect ratio — do not reference external image URLs unless the user provides them.

---

## Step 5 — Output

Output the complete HTML file contents directly in a code block tagged `html`.

After the code block, add a brief note (2–4 lines) stating:
- Which template was used and why
- Any style overrides that were applied
- Any substitutions made (e.g. font swap from proprietary to Google Fonts)
