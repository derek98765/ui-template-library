---
name: styles-extractor
description: >
  Extract design tokens and a screenshot from any website URL. Trigger this skill whenever the user wants to extract styles, analyze a design system, get design tokens, or reverse-engineer the visual design of a website. Uses source code inspection as the primary method, with Chrome extension JavaScript execution as a fallback for dynamically rendered styles. Outputs a structured JSON token file and a screenshot — no UI replication.
---

# Styles Extractor Skill

Analyze any given website URL and extract a comprehensive design token inventory. Uses source code inspection as the primary method, with Chrome extension JavaScript execution as a fallback for dynamically rendered styles.

**Requires:** Claude in Chrome extension (for live site analysis and fallback JS extraction)

## Trigger

Invoke this skill when the user says:
- "extract styles from [URL]"
- "analyze the design of [URL]"
- "get the design tokens from [URL]"
- "what styles does [URL] use?"
- "reverse engineer the design system of [URL]"
- "inspect [URL]"

## Outputs

Only two files are produced:
- **Screenshot:** `/Users/derekfung/.claude/skills/styles-extractor/references/<industry>/<sitename>/<sitename>.png`
- **JSON tokens:** `/Users/derekfung/.claude/skills/styles-extractor/references/<industry>/<sitename>/<sitename>.json`

---

## Phase 1 — Source Code Inspection (Primary)

### 1.1 Fetch HTML & Discover CSS

Use `WebFetch` to retrieve the page HTML. Then:

1. Find all `<link rel="stylesheet">` tags and fetch each CSS file with `WebFetch`
2. Extract all inline `<style>` blocks from the HTML
3. Check for font imports:
   - `<link href="fonts.googleapis.com/...">` tags
   - `@import url(...)` inside style blocks
4. Look for framework signals:
   - CSS custom properties in `:root` or `[data-theme]` / `[data-color-scheme]`
   - Tailwind utility class patterns (`bg-`, `text-`, `rounded-`, `shadow-`, `p-`, `m-`)
   - CSS-in-JS output (`<style data-emotion>`, `<style data-styled>`, styled-components hashes)

### 1.2 Extract Design Tokens from CSS

Parse the fetched CSS and extract all values by category (see **Extraction Categories** below). For each value:
- Record the raw CSS value and a normalized form (e.g., hex for colors, px for sizes)
- Note the source: `css-custom-property`, `class-name`, `inline-style`, or `tailwind`
- Flag values that cannot be confirmed directly as `[inferred]`

Prioritize in this order:
1. **CSS custom properties** — `:root { --color-primary: ... }` is the most reliable source
2. **Tailwind classes** — map utility classes to the default Tailwind palette if no config override is found
3. **Repeated patterns** — if multiple elements share `padding: 16px 24px`, treat it as a spacing token
4. **Interactive states** — hover/focus/active styles often reveal accent and semantic colors

---

## Phase 2 — JavaScript Fallback (When Source Inspection Is Insufficient)

**Skip this phase entirely if Phase 1 yielded colors, typography, and spacing tokens.** Only proceed here if:
- CSS files are minified/obfuscated with no readable token values, OR
- Key token categories (colors, fonts) are completely missing from source CSS, OR
- Styles are clearly injected at runtime (CSS-in-JS with hashed class names only)

Do NOT run JS snippets proactively or "just to double-check". If Phase 1 produced a reasonably complete token set, go directly to Phase 3.

When Phase 2 is needed, navigate to the URL in Chrome first:

```
navigate → [user's URL]
```

### 2.1 Extract Computed Design Tokens

```javascript
// Extract design tokens from computed styles
const allElements = document.querySelectorAll('*');
const colors = new Set();
const fonts = new Set();
const fontSizes = new Set();
const radii = new Set();
const shadows = new Set();

allElements.forEach(el => {
  const s = window.getComputedStyle(el);
  if (s.color) colors.add(s.color);
  if (s.backgroundColor && s.backgroundColor !== 'rgba(0, 0, 0, 0)') colors.add(s.backgroundColor);
  if (s.fontFamily) fonts.add(s.fontFamily.split(',')[0].trim());
  if (s.fontSize) fontSizes.add(s.fontSize);
  if (s.borderRadius && s.borderRadius !== '0px') radii.add(s.borderRadius);
  if (s.boxShadow && s.boxShadow !== 'none') shadows.add(s.boxShadow);
});

const body = document.body;
const computed = window.getComputedStyle(body);
JSON.stringify({
  colors: [...colors].slice(0, 40),
  fonts: [...fonts].slice(0, 10),
  fontSizes: [...fontSizes].slice(0, 20),
  borderRadii: [...radii].slice(0, 10),
  boxShadows: [...shadows].slice(0, 8),
  bodyBg: computed.backgroundColor,
  bodyColor: computed.color,
  bodyFont: computed.fontFamily
}, null, 2)
```

### 2.2 Extract CSS Custom Properties at Runtime

```javascript
// Extract CSS variables / design tokens from :root
const sheets = [...document.styleSheets];
const vars = {};
try {
  sheets.forEach(sheet => {
    try {
      [...sheet.cssRules].forEach(rule => {
        if (rule.selectorText === ':root' || rule.selectorText === 'html') {
          const style = rule.style;
          for (let i = 0; i < style.length; i++) {
            const prop = style[i];
            if (prop.startsWith('--')) {
              vars[prop] = style.getPropertyValue(prop).trim();
            }
          }
        }
      });
    } catch(e) {}
  });
} catch(e) {}
JSON.stringify(vars, null, 2)
```

### 2.3 Extract Loaded Fonts

```javascript
// Check loaded fonts and font link tags
const fonts = [];
if (document.fonts) {
  document.fonts.forEach(f => {
    fonts.push({ family: f.family, style: f.style, weight: f.weight, status: f.status });
  });
}
const fontLinks = [...document.querySelectorAll('link[href*="font"], link[href*="typekit"]')]
  .map(l => l.href);
const fontImports = [...document.querySelectorAll('style')]
  .map(s => s.textContent)
  .join('\n')
  .match(/@import url\([^)]+\)/g) || [];
JSON.stringify({ loadedFonts: fonts.slice(0, 20), fontLinks, fontImports }, null, 2)
```

### 2.4 Extract Breakpoints

```javascript
// Extract media query breakpoints
const breakpoints = new Set();
try {
  [...document.styleSheets].forEach(sheet => {
    try {
      [...sheet.cssRules].forEach(rule => {
        if (rule instanceof CSSMediaRule) {
          breakpoints.add(rule.conditionText);
        }
      });
    } catch(e) {}
  });
} catch(e) {}
JSON.stringify([...breakpoints].slice(0, 15))
```

---

## Phase 3 — Full-Page Screenshot

Capture a **full-page** screenshot at 1440px wide. A fixed 900px height only captures the viewport — always measure the actual document height first.

### Step 1 — Measure the full page height

If the Chrome extension is open and the page is already loaded (from Phase 2), run:

```javascript
Math.max(document.body.scrollHeight, document.documentElement.scrollHeight)
```

Store the result as `FULL_HEIGHT`. If the Chrome extension is not in use, skip to Step 2 and use a sensible default (e.g. `8000`).

### Step 2 — Take the screenshot with Chrome headless CLI

Pass `FULL_HEIGHT` as the window height so Chrome renders and captures the entire page:

```bash
FULL_HEIGHT=<value from Step 1>   # e.g. 6240
OUTPUT_PATH="/Users/derekfung/.claude/skills/styles-extractor/references/<industry>/<sitename>/<sitename>.png"
URL="https://example.com"

/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --headless=new \
  --screenshot="$OUTPUT_PATH" \
  --window-size=1440,$FULL_HEIGHT \
  --hide-scrollbars \
  --virtual-time-budget=5000 \
  "$URL" 2>&1 | tail -3
```

If Chrome is not at that path, try the system commands:
```bash
google-chrome  --headless=new --screenshot="$OUTPUT_PATH" --window-size=1440,$FULL_HEIGHT --hide-scrollbars "$URL"
chromium-browser --headless=new --screenshot="$OUTPUT_PATH" --window-size=1440,$FULL_HEIGHT --hide-scrollbars "$URL"
```

### Fallback — Playwright or Puppeteer (if Chrome headless fails)

```bash
# Option A — Playwright (has --full-page built-in)
npx playwright@latest screenshot --browser chromium --full-page \
  --viewport-size="1440,900" "$URL" "$OUTPUT_PATH"

# Option B — Puppeteer inline Node script
node - << 'EOF'
const puppeteer = require('puppeteer');
(async () => {
  const b = await puppeteer.launch({ args: ['--no-sandbox'] });
  const p = await b.newPage();
  await p.setViewport({ width: 1440, height: 900 });
  await p.goto('URL', { waitUntil: 'networkidle2', timeout: 30000 });
  await p.screenshot({ path: 'OUTPUT_PATH', fullPage: true });
  await b.close();
})().catch(console.error);
EOF
```

Try each fallback in order. If all fail, record the error message in `_meta.screenshotError` and continue without the screenshot.

---

## Phase 4 — Classify Industry

Based on the site's content, visual design, and brand context, assign it to exactly one of these 7 categories:

| # | Folder name   | Typical sites                                               |
|---|---------------|--------------------------------------------------------------|
| 1 | `Finance`     | Banking, payments, investing, trading, insurance, crypto     |
| 2 | `Tech & SaaS` | Developer tools, productivity, APIs, cloud platforms, B2B    |
| 3 | `Healthcare`  | Medical, wellness, pharma, health tech, telemedicine         |
| 4 | `E-commerce`  | Online retail, marketplaces, DTC brands, subscription boxes  |
| 5 | `Services`    | Agencies, consulting, legal, real estate, professional svcs  |
| 6 | `Creative`    | Design tools, media, entertainment, portfolios, publishing   |
| 7 | `Lifestyle`   | Food, travel, fitness, fashion, beauty, home                 |

Record the chosen category in `brand.industry` (use the exact folder name).

---

## Phase 5 — Save Outputs

Derive the site name from the root domain (lowercase, no TLD suffix):
- `stripe.com` → `stripe`
- `linear.app` → `linear`
- `app.vercel.com` → `vercel`

Create the output folder:
```bash
mkdir -p "/Users/derekfung/.claude/skills/styles-extractor/references/<industry>/<sitename>"
```

Save both files:
- **Screenshot:** `/Users/derekfung/.claude/skills/styles-extractor/references/<industry>/<sitename>/<sitename>.png`
- **JSON tokens:** `/Users/derekfung/.claude/skills/styles-extractor/references/<industry>/<sitename>/<sitename>.json`

---

## Extraction Categories

### 1. Color
- **Primary** — main brand/action color (buttons, links, highlights)
- **Secondary** — supporting brand color
- **Accent** — tertiary or highlight color
- **Semantic** — success (green), error (red), warning (yellow/orange), info (blue)
- **Neutral / Gray scale** — full ramp from lightest to darkest (gray-50 → gray-900)
- **Background & Surface** — page bg, card bg, sidebar bg, overlay bg
- **Text colors** — primary, secondary, muted/placeholder, on-dark

### 2. Typography
- **Font families** — heading, body, monospace/code
- **Font size scale** — xs, sm, base, md, lg, xl, 2xl, 3xl, 4xl
- **Font weights** — thin (100) through black (900)
- **Line heights** — tight, snug, normal, relaxed, loose
- **Letter spacing** — tighter through widest
- **Heading hierarchy** — H1–H6: size, weight, line-height, letter-spacing, color

### 3. Spacing & Layout
- **Spacing scale** — base unit (usually 4px or 8px) and full scale in px
- **Container max-widths** — sm through 2xl
- **Grid system** — columns, gutter, margin
- **Section padding** — common vertical/horizontal padding for page sections

### 4. Border & Shape
- **Border radius scale** — none through full/pill
- **Border widths** — thin (1px), medium (2px), thick (4px)
- **Border colors** — default, muted, strong, focus ring
- **Common border styles** — solid, dashed, dotted

### 5. Shadows & Elevation
- **Shadow scale** — sm, md, lg, xl
- **Elevation mapping** — card, dropdown, modal, tooltip

### 6. Components (tokens only, not markup)
- **Button** — primary, secondary, ghost; sizes sm/md/lg
- **Input** — bg, border, radius, padding, placeholder color, focus state
- **Card** — bg, border, radius, padding, shadow
- **Badge/Tag** — bg, text color, radius, padding, font size/weight
- **Navbar** — bg, height, border-bottom, item color, active color/style

### 7. Iconography & Imagery
- **Icon style** — outline, filled, rounded, sharp
- **Icon sizes** — sm (16px), md (20px), lg (24px), xl (32px)
- **Image type** — identify the dominant type of imagery used across the site:
  - `illustration` — vector/drawn artwork (flat, geometric, or stylised)
  - `photography` — real-world photographs
  - `mixed` — both illustrations and photos used together
  - `none` — no significant imagery, icon/text-only
- **Image style** — describe the visual rendering style of the images:
  - `photography-real` — actual photographs of real people, places, or objects
  - `photography-lifestyle` — styled/staged lifestyle photography (people in context)
  - `3d-realistic` — photorealistic 3D renders (products, environments)
  - `3d-clay` — clay/matte 3D renders with soft lighting and pastel tones
  - `illustration-2d-flat` — flat vector illustrations, minimal shading
  - `illustration-2d-geometric` — abstract geometric shapes and patterns
  - `illustration-handdrawn` — hand-drawn or sketch-style artwork
  - `illustration-3d` — 3D-styled cartoon or stylised illustrations
  - `mixed` — multiple styles used across the site
- **Human presence** — whether images include people:
  - `yes-prominent` — people are a central visual element (faces, full body shots)
  - `yes-incidental` — people appear but are not the main focus
  - `no` — no people depicted; product, abstract, or environment imagery only
- **Image treatment** — border radius, common aspect ratios, overlay/filter, hover effects

### 8. Brand Meta
- **Tone** — minimal, clean, bold, playful, friendly, professional, premium, technical, trustworthy, energetic
- **Visual language** — dark mode, light mode, flat 2D, glassmorphism, neumorphism, brutalist, editorial, data-dense
- **Industry** — chosen category from Phase 4
- **Notes** — free-text observations about the visual identity

---

## Output Format

Save as `<sitename>.json` inside the output folder. Populate every field. Use `null` for genuinely undetectable values. Append `" [inferred]"` to any string value that could not be confirmed from CSS.

```json
{
  "_meta": {
    "site": "Site Name",
    "url": "https://example.com",
    "date": "YYYY-MM-DD",
    "source": ["css-custom-properties", "tailwind", "inline-styles", "inferred"],
    "screenshotPath": "/Users/derekfung/.claude/skills/styles-extractor/references/<industry>/<sitename>/<sitename>.png",
    "screenshotError": null
  },

  "color": {
    "brand": {
      "primary": "#XXXXXX",
      "secondary": "#XXXXXX",
      "accent": "#XXXXXX"
    },
    "semantic": {
      "success": "#XXXXXX",
      "error": "#XXXXXX",
      "warning": "#XXXXXX",
      "info": "#XXXXXX"
    },
    "neutral": {
      "50":  "#XXXXXX",
      "100": "#XXXXXX",
      "200": "#XXXXXX",
      "300": "#XXXXXX",
      "400": "#XXXXXX",
      "500": "#XXXXXX",
      "600": "#XXXXXX",
      "700": "#XXXXXX",
      "800": "#XXXXXX",
      "900": "#XXXXXX"
    },
    "background": {
      "page":    "#XXXXXX",
      "card":    "#XXXXXX",
      "sidebar": "#XXXXXX",
      "overlay": "#XXXXXX"
    },
    "text": {
      "primary":   "#XXXXXX",
      "secondary": "#XXXXXX",
      "muted":     "#XXXXXX",
      "onDark":    "#XXXXXX"
    }
  },

  "typography": {
    "fontFamilies": {
      "heading": "Font Name, fallback",
      "body":    "Font Name, fallback",
      "mono":    "Font Name, fallback"
    },
    "fontSizes": {
      "xs":   "12px",
      "sm":   "14px",
      "base": "16px",
      "md":   "18px",
      "lg":   "20px",
      "xl":   "24px",
      "2xl":  "30px",
      "3xl":  "36px",
      "4xl":  "48px"
    },
    "fontWeights": {
      "thin":      100,
      "light":     300,
      "regular":   400,
      "medium":    500,
      "semibold":  600,
      "bold":      700,
      "extrabold": 800,
      "black":     900
    },
    "lineHeights": {
      "tight":   1.25,
      "snug":    1.375,
      "normal":  1.5,
      "relaxed": 1.625,
      "loose":   2.0
    },
    "letterSpacing": {
      "tighter": "-0.05em",
      "tight":   "-0.025em",
      "normal":  "0",
      "wide":    "0.025em",
      "wider":   "0.05em",
      "widest":  "0.1em"
    },
    "headings": {
      "h1": { "fontSize": "48px", "fontWeight": 700, "lineHeight": 1.1, "letterSpacing": "-0.02em", "color": "#XXXXXX" },
      "h2": { "fontSize": "36px", "fontWeight": 700, "lineHeight": 1.2, "letterSpacing": "-0.01em", "color": "#XXXXXX" },
      "h3": { "fontSize": "30px", "fontWeight": 600, "lineHeight": 1.3, "letterSpacing": "0",       "color": "#XXXXXX" },
      "h4": { "fontSize": "24px", "fontWeight": 600, "lineHeight": 1.4, "letterSpacing": "0",       "color": "#XXXXXX" },
      "h5": { "fontSize": "20px", "fontWeight": 600, "lineHeight": 1.4, "letterSpacing": "0",       "color": "#XXXXXX" },
      "h6": { "fontSize": "16px", "fontWeight": 600, "lineHeight": 1.5, "letterSpacing": "0",       "color": "#XXXXXX" }
    }
  },

  "spacing": {
    "baseUnit": "4px",
    "scale": {
      "0":  "0px",
      "1":  "4px",
      "2":  "8px",
      "3":  "12px",
      "4":  "16px",
      "5":  "20px",
      "6":  "24px",
      "8":  "32px",
      "10": "40px",
      "12": "48px",
      "16": "64px",
      "20": "80px",
      "24": "96px"
    },
    "containers": {
      "sm":  "640px",
      "md":  "768px",
      "lg":  "1024px",
      "xl":  "1280px",
      "2xl": "1536px"
    },
    "grid": {
      "columns": 12,
      "gutter":  "24px",
      "margin":  "24px"
    },
    "sectionPadding": {
      "vertical":   "80px",
      "horizontal": "24px"
    }
  },

  "border": {
    "radius": {
      "none": "0",
      "sm":   "2px",
      "base": "4px",
      "md":   "6px",
      "lg":   "8px",
      "xl":   "12px",
      "2xl":  "16px",
      "full": "9999px"
    },
    "widths": {
      "thin":   "1px",
      "medium": "2px",
      "thick":  "4px"
    },
    "colors": {
      "default": "#XXXXXX",
      "muted":   "#XXXXXX",
      "strong":  "#XXXXXX",
      "focus":   "#XXXXXX"
    },
    "styles": ["solid", "dashed", "dotted"]
  },

  "shadows": {
    "sm": "0 1px 2px rgba(0,0,0,0.05)",
    "md": "0 4px 6px rgba(0,0,0,0.1)",
    "lg": "0 10px 15px rgba(0,0,0,0.15)",
    "xl": "0 20px 25px rgba(0,0,0,0.2)"
  },
  "elevation": {
    "card":     "sm",
    "dropdown": "md",
    "modal":    "lg",
    "tooltip":  "sm"
  },

  "components": {
    "button": {
      "primary": {
        "background":   "#XXXXXX",
        "text":         "#XXXXXX",
        "border":       "none",
        "borderRadius": "6px",
        "paddingX":     "16px",
        "paddingY":     "8px",
        "fontWeight":   600,
        "fontSize":     "14px"
      },
      "secondary": {
        "background":   "#XXXXXX",
        "text":         "#XXXXXX",
        "border":       "1px solid #XXXXXX",
        "borderRadius": "6px",
        "paddingX":     "16px",
        "paddingY":     "8px",
        "fontWeight":   600,
        "fontSize":     "14px"
      },
      "ghost": {
        "background":   "transparent",
        "text":         "#XXXXXX",
        "border":       "1px solid #XXXXXX",
        "borderRadius": "6px",
        "paddingX":     "16px",
        "paddingY":     "8px",
        "fontWeight":   600,
        "fontSize":     "14px"
      },
      "sizes": {
        "sm": { "height": "32px", "paddingX": "12px", "fontSize": "12px" },
        "md": { "height": "40px", "paddingX": "16px", "fontSize": "14px" },
        "lg": { "height": "48px", "paddingX": "24px", "fontSize": "16px" }
      }
    },
    "input": {
      "background":       "#XXXXXX",
      "borderColor":      "#XXXXXX",
      "borderRadius":     "6px",
      "padding":          "8px 12px",
      "fontSize":         "14px",
      "placeholderColor": "#XXXXXX",
      "focus": {
        "borderColor": "#XXXXXX",
        "ringColor":   "#XXXXXX",
        "ringWidth":   "2px"
      }
    },
    "card": {
      "background":   "#XXXXXX",
      "border":       "1px solid #XXXXXX",
      "borderRadius": "8px",
      "padding":      "24px",
      "shadow":       "md"
    },
    "badge": {
      "background":   "#XXXXXX",
      "text":         "#XXXXXX",
      "borderRadius": "9999px",
      "padding":      "2px 8px",
      "fontSize":     "12px",
      "fontWeight":   500
    },
    "navbar": {
      "background":   "#XXXXXX",
      "height":       "64px",
      "borderBottom": "1px solid #XXXXXX",
      "itemColor":    "#XXXXXX",
      "activeColor":  "#XXXXXX",
      "activeStyle":  "underline | background | border-left"
    }
  },

  "iconography": {
    "style": "outline | filled | rounded | sharp",
    "sizes": {
      "sm": "16px",
      "md": "20px",
      "lg": "24px",
      "xl": "32px"
    }
  },
  "imagery": {
    "type":          "illustration | photography | mixed | none",
    "style":         "photography-real | photography-lifestyle | 3d-realistic | 3d-clay | illustration-2d-flat | illustration-2d-geometric | illustration-handdrawn | illustration-3d | mixed",
    "humanPresence": "yes-prominent | yes-incidental | no",
    "borderRadius":  "8px",
    "aspectRatios":  ["16:9", "4:3", "1:1"],
    "treatments":    ["dark overlay", "rounded corners"],
    "notes":         "Optional observations — e.g. 'hero uses lifestyle photography of diverse users; feature sections use 2D flat icons; product screenshots on dark backgrounds'"
  },

  "brand": {
    "tone":           ["minimal", "professional", "trustworthy"],
    "visualLanguage": ["dark mode", "flat 2D", "data-dense"],
    "industry":       "Finance",
    "notes":          "Optional free-text observations about the visual identity."
  }
}
```

---

## Tips for Accurate Extraction

- **Prioritize CSS custom properties** — `:root { --color-primary: ... }` is the most reliable source.
- **Check Tailwind** — map utility classes like `bg-blue-600` or `text-gray-900` to the default Tailwind palette when no custom config is found.
- **Check Google Fonts links** — `fonts.googleapis.com/css2?family=Inter` directly reveals the font family.
- **Infer spacing from patterns** — if multiple elements share `padding: 16px 24px`, treat it as a spacing token.
- **Look at interactive states** — hover/focus/active styles often reveal accent and semantic colors not used in the base layout.
- **Dark mode** — check for `@media (prefers-color-scheme: dark)` or `[data-theme="dark"]` selectors.
- **Use JS fallback selectively** — only invoke Chrome extension JavaScript when source inspection leaves significant gaps; do not run all JS snippets by default.
- **File naming** — always derive the filename from the root domain only: `stripe.com` → `stripe`, `app.linear.app` → `linear`.
- **Output folder** — save to `/Users/derekfung/.claude/skills/styles-extractor/references/<industry>/<sitename>/`. Valid industry folder names: `Finance`, `Tech & SaaS`, `Healthcare`, `E-commerce`, `Services`, `Creative`, `Lifestyle`. Quote the path when creating it to handle spaces (e.g., `"Tech & SaaS"`).
- **Full-page screenshot** — always measure `document.documentElement.scrollHeight` first, then pass that value as the Chrome headless `--window-size` height (e.g. `--window-size=1440,6240`). A fixed 900px height only captures the viewport. PNG format preferred over JPG.
