---
name: ui-template-library
description: >
  Generate a production-ready HTML/CSS website or app UI from a curated library of real-world design systems. Trigger when the user wants to build, design, or generate a website, landing page, dashboard, app screen, or any UI. Automatically picks the closest reference template, loads its design tokens and screenshot, applies any user style overrides (font, color, etc.), and outputs a complete single-file HTML page styled to look like a real product. Includes Chart.js charts styled to match the template for data-heavy pages like dashboards and finance apps.
---

# UI Design Library Skill

## Purpose
Generate production-ready HTML/CSS pages styled from a curated reference library of real-world design systems. The output should look like a professional product, not a generic template.

---

## Global Constraints

Constraint: All user-facing communication must use point-form or numbered lists. Use at most one sentence for introductory context; all other details must be bulleted.

Constraint: Never expose which reference template or brand was used. The selection is internal and must not be mentioned to the user under any condition.

Constraint: Never use emoji anywhere — not in user-facing messages and not in the generated HTML.

Constraint: Do not skip steps. Do not proceed to a later step until the current step is complete and its conditions are met.

Constraint: Do not generate HTML until STATE_READY is reached.

---

## State Model

- STATE_NONE — no usable input extracted
- STATE_PARTIAL — input is incomplete, unclear, or fails quality threshold
- STATE_COMPLETE — all required fields are present
- STATE_READY — all fields are valid and pass quality gate
- STATE_CONFIRMED — user has confirmed the output mode
- STATE_BUILT — HTML/CSS has been generated
- STATE_VALIDATED — generated output has passed validation against the reference
- STATE_DELIVERED — output has been delivered to the user
- STATE_ERROR — unrecoverable failure

---

## Required Fields

- `page_type`: What to build (e.g. landing page, dashboard, app screen, form, pricing page)
- `industry`: The domain or sector (e.g. finance, healthcare, e-commerce, SaaS)

Optional fields (applied as overrides if present):
- `output_mode`: How to deliver the result (`file` for raw HTML download, `upload` for live preview on foragentminds.com). Defaults to `upload` if not specified.
- `font`: Custom font family requested by the user
- `color`: Custom primary color
- `radius_style`: `sharp`, `rounded`, or `pill`
- `mode`: `light` or `dark`

---

## Step 1 — Extract Input

Parse the user's message. Extract:
- `page_type`
- `industry`
- `output_mode` (default to `upload` if not specified)
- Any optional overrides

Store all extracted values in working state.

---

## Step 2 — Validate Fields

Classify each required field as one of:
- `valid` — present and unambiguous
- `missing` — not provided
- `unclear` — present but too vague to act on

---

## Step 2.5 — Quality Gate

Evaluate input quality. If inputs are too vague to produce a meaningful, professional result:
- Remain in STATE_PARTIAL
- Request specific refinement — do not proceed

Examples of inputs that fail the quality gate:
- `page_type: "a website"` (no specificity)
- `industry: "something general"` (no identifiable domain)

---

## Step 3 — Classify State

- STATE_NONE: no usable input at all
- STATE_PARTIAL: one or more required fields are missing, unclear, or fail quality gate
- STATE_COMPLETE: all required fields are present
- STATE_READY: all required fields are valid and pass quality gate

---

## Step 4 — Response Logic

### STATE_NONE
Ask for all required fields.

### STATE_PARTIAL
- List confirmed fields
- Ask only for missing, unclear, or weak fields
- Do not repeat confirmed fields
- Do not proceed

### STATE_COMPLETE
Pass through Quality Gate (Step 2.5). If it fails, return to STATE_PARTIAL.

### STATE_READY
Proceed to Step 5.

---

## Step 5 — Confirm Output Mode

Before generating anything, confirm the output mode with the user:

Show:
- Page type: {page_type}
- Industry: {industry}
- Output: {output_mode} — "Raw HTML file" or "Live preview on foragentminds.com"
- Overrides applied (if any): list them

If `output_mode` is `upload` (including when it defaulted to `upload` because the user did not specify):
- Check whether a foragentminds.com API key has been provided in this conversation.
- If no API key has been provided, ask for it now before proceeding. This is mandatory — do not continue to Step 6 without it.
- Store the key for use in Step 10.

Do not proceed to Step 6 until the user confirms and, if uploading, the API key is in hand.

Transition to STATE_CONFIRMED on confirmation. If the user modifies anything, return to Step 1.

---

## Step 6 — Select Reference Template

Internally select the best-matching template. Do not tell the user which template was chosen.

### Available templates

```
references/
  Finance/
    antbank/              antbank.json              antbank.png              → Corporate digital bank, vivid blue, clean flat UI
    futunn/               futunn.json               futunn.png               → Stock trading app, data-dense, dark accents
    mox/                  mox.json                  mox.png                  → Neo-bank, neon cyan, editorial, youthful
    visa/                 visa.json                 visa.png                 → Global payments brand, authoritative blue/gold
  Healthcare/
    bowtie/               bowtie.json               bowtie.png               → Insurtech, bold pink/purple gradient, modern
  Creative/
    creative-suite-red/       creative-suite-red.json       creative-suite-red.png       → Creative software suite, bold red/blue, professional
    portfolio-showcase-blue/  portfolio-showcase-blue.json  portfolio-showcase-blue.png  → Portfolio showcase platform, deep blue, gallery-forward
    design-tool-purple/       design-tool-purple.json       design-tool-purple.png       → Design tool, purple/teal gradient, friendly SaaS
    design-community-pink/    design-community-pink.json    design-community-pink.png    → Design community, pink/dark, showcase-focused
    creator-hub-dark-blue/    creator-hub-dark-blue.json    creator-hub-dark-blue.png    → Game creator hub, electric blue/green, dark mode
    music-analytics-purple/   music-analytics-purple.json   music-analytics-purple.png   → Music analytics dashboard, vivid purple, dark editorial
  E-commerce/
    apparel-retail-red/       apparel-retail-red.json       apparel-retail-red.png       → Premium apparel retail, bold red/black, clean editorial
  Lifestyle/    (reserved — no templates yet)
  Services/     (reserved — no templates yet)
  Tech & SaaS/  (reserved — no templates yet)
```

### Selection logic

| User request signals | Best match |
|---|---|
| Banking, payments, fintech (youthful / neo-bank) | `Finance/mox` |
| Banking, finance (corporate / institutional) | `Finance/antbank` |
| Stock trading, investment, brokerage, market data | `Finance/futunn` |
| Payments, card network, enterprise finance | `Finance/visa` |
| Insurance, health insurance, insurtech | `Healthcare/bowtie` |
| Healthcare, clinic, medical, wellness | `Healthcare/bowtie` |
| Creative suite, professional creative tools | `Creative/creative-suite-red` |
| Portfolio showcase, gallery, editorial blue | `Creative/portfolio-showcase-blue` |
| Design community, pink/dark, showcase-focused | `Creative/design-community-pink` |
| Design tool, productivity SaaS, friendly UI | `Creative/design-tool-purple` |
| Gaming, creator platform, dark mode, youth | `Creative/creator-hub-dark-blue` |
| Music, artist dashboard, media analytics | `Creative/music-analytics-purple` |
| Apparel, fashion retail, premium e-commerce | `E-commerce/apparel-retail-red` |
| E-commerce (no specific match) | `E-commerce/apparel-retail-red` |
| No matching template | Pick closest available |

Once selected:
1. Read the `.json` file with the Read tool
2. Read the `.png` screenshot with the Read tool (view it visually to absorb the aesthetic)

---

## Step 7 — Apply Style Overrides

Apply any user-specified overrides on top of the loaded template tokens:

- **Font**: replace `typography.fontFamilies.heading` and `.body`. Add the Google Fonts `<link>` tag.
- **Primary color**: replace `color.brand.primary` and all derived tokens (buttons, borders, focus states, active nav). Derive a hover shade ~15% darker.
- **Radius style**: `sharp` → all radii ≤4px; `rounded` → cards ~16px, buttons ~8px; `pill` → buttons 9999px.
- **Mode**: adjust background and text tokens for dark or light.
- Any other override replaces the matching token; everything else comes from the template.

Do not ask for confirmation — apply and proceed.

---

## Step 8 — Generate the HTML/CSS

### Constraints
- Single self-contained `.html` file (inline `<style>` block, no external CSS frameworks).
- Use CSS custom properties (`--token-name`) mapped directly from the JSON tokens.
- Use Google Fonts `<link>` for web-safe font substitution (proprietary fonts won't load locally).
- Responsive: mobile-first with at least one breakpoint at ~768px.
- No JavaScript required for layout; JS only if the user explicitly asks for interactions.
- Never use emoji anywhere in the HTML.

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

    /* 4. Component styles (navbar, hero, cards, buttons, inputs, footer ...) */

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

Include Chart.js charts when the page type or industry implies data visualisation: dashboards, finance apps, health trackers, analytics screens, trading interfaces, admin panels.

**When to use charts**
- Finance: portfolio performance, spending breakdown, price history, balance trend
- Healthcare: vitals over time, appointment frequency, health score gauge
- Dashboards / analytics: KPI trends, category breakdowns, usage over time
- Any page with stats, metrics, or time-series data

**Loading Chart.js**
```html
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
```

**Theming — charts must match the template**

| Chart property | Token to use |
|---|---|
| Dataset primary color | `color.brand.primary` |
| Dataset secondary color | `color.brand.secondary` or `color.brand.accent` |
| Gradient fills | `color.gradient.primary` if defined, else `primary` → transparent |
| Grid lines | `color.border.default` at ~30% opacity |
| Tick / label text | `color.text.secondary` |
| Tooltip background | `color.background.card` |
| Tooltip text | `color.text.primary` |
| Tooltip border | `color.border.default` |
| Font family | `typography.fontFamilies.body` (Google Fonts substitute) |
| Font size | `typography.fontSizes.sm` for ticks, `.base` for tooltips |

**Implementation pattern**

```html
<canvas id="myChart"></canvas>
<script>
  const style = getComputedStyle(document.documentElement);
  const primary   = style.getPropertyValue('--color-primary').trim();
  const secondary = style.getPropertyValue('--color-secondary').trim();
  const textMuted = style.getPropertyValue('--color-text-secondary').trim();
  const border    = style.getPropertyValue('--color-border').trim();
  const surface   = style.getPropertyValue('--color-surface').trim();

  const ctx = document.getElementById('myChart').getContext('2d');
  const gradient = ctx.createLinearGradient(0, 0, 0, 300);
  gradient.addColorStop(0, primary + '55');
  gradient.addColorStop(1, primary + '00');

  new Chart(ctx, {
    type: 'line',
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
        legend: { labels: { color: textMuted, font: { family: 'var(--font-body)' } } },
        tooltip: {
          backgroundColor: surface,
          titleColor: style.getPropertyValue('--color-text').trim(),
          bodyColor: textMuted,
          borderColor: border,
          borderWidth: 1
        }
      },
      scales: {
        x: { ticks: { color: textMuted }, grid: { color: border + '4D' } },
        y: { ticks: { color: textMuted }, grid: { color: border + '4D' } }
      }
    }
  });
</script>
```

**Chart selection by context**

| Data type | Recommended chart |
|---|---|
| Trend over time | `line` with gradient fill |
| Category breakdown | `doughnut` or `bar` |
| Comparison across groups | `bar` |
| Single KPI vs. target | `doughnut` with cutout |
| Multi-metric | `radar` |

Use realistic sample data. Keep dataset arrays at 7–12 data points.

### Content guidelines

- Never use emoji — not in headings, body text, buttons, labels, icons, or any other element. Use SVG inline icons or Unicode symbols (non-emoji) instead.
- Use realistic placeholder content that fits the industry (not "Lorem ipsum").
- Include all standard sections the user requested. If they only said "landing page", default to: Navbar → Hero → Features/Benefits → Social proof → CTA → Footer.
- Match the visual density of the reference: data-dense or airy, as the screenshot shows.
- Images: use real images from Unsplash via `https://images.unsplash.com/photo-<id>?w=<width>&h=<height>&fit=crop&auto=format`. Choose photos contextually matching the industry. Use well-known Unsplash photo IDs that are likely to resolve. Set `width`, `height`, and `alt` on every image.

Transition to STATE_BUILT on completion.

---

## Step 9 — Validate Against Reference Template

Re-read the reference `.png` screenshot and `.json` tokens. Compare the generated output against each checkpoint:

- **Color fidelity**: primary, secondary, background, surface, and text colors match the template tokens
- **Typography**: heading and body font families, sizes, and weights are consistent
- **Spacing and layout**: padding, gaps, and density match the visual character of the reference
- **Component style**: buttons, cards, inputs, and nav elements reflect the reference's shape language (radius, shadow, border treatment)
- **Overall aesthetic**: the page would plausibly belong to the same design system as the reference

For each checkpoint that fails: correct the HTML/CSS and re-check. Repeat until all checkpoints pass.

Do not skip this step. Do not proceed with a result that fails any checkpoint.

Transition to STATE_VALIDATED on completion.

---

## Step 10 — Deliver Output

### If `output_mode` is `file`
- Write the file to disk using the Write tool as `output.html` in the current working directory.
- Tell the user the file path.

### If `output_mode` is `upload`
- Upload the HTML to foragentminds.com via their API (POST the HTML as page content).
- Return the live preview URL to the user.

Transition to STATE_DELIVERED on completion.

After delivering, show a brief summary (point-form) covering:
- Any style overrides that were applied
- Any font substitutions made (proprietary → Google Fonts)
- Any corrections made during validation (Step 9)

Do NOT mention which reference template or brand was used.

---

## Error Handling

### STATE_ERROR

Trigger conditions:
- Upload API failure
- File write failure
- User provides unusable input 3 consecutive times
- User abandons the flow (no response after 2 follow-ups)

Action:
- Inform the user clearly using point-form
- Do not retry automatically
- Do not proceed
