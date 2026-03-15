# ADAPT.md — Porting Google Cloud Run / AI Studio Simulations to PanPhy Labs

This guide documents the complete process of adapting a simulation originally built with a modern framework (React, Vite, Tailwind, etc.) for Google Cloud Run or Google AI Studio into a self-contained, offline-capable page for the PanPhy Labs GitHub Pages site. It was written during the adaptation of **Physical Pendulum Explorer** and is designed to be reused for every future simulation you port.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Before You Start — Understand Both Worlds](#2-before-you-start--understand-both-worlds)
3. [Step 1: Audit the Source Simulation](#3-step-1-audit-the-source-simulation)
4. [Step 2: Extract the Core Simulation Logic](#4-step-2-extract-the-core-simulation-logic)
5. [Step 3: Rewrite as a Single-File Vanilla HTML Page](#5-step-3-rewrite-as-a-single-file-vanilla-html-page)
6. [Step 4: Apply the PanPhy Labs Design System](#6-step-4-apply-the-panphy-labs-design-system)
7. [Step 5: Integrate with the Site Infrastructure](#7-step-5-integrate-with-the-site-infrastructure)
8. [Step 6: Verify Everything Works](#8-step-6-verify-everything-works)
9. [Quick Reference Checklist](#9-quick-reference-checklist)
10. [Common Adaptation Patterns](#10-common-adaptation-patterns)
11. [Appendix A: CSS Variable Reference](#appendix-a-css-variable-reference)
12. [Appendix B: The Physical Pendulum Case Study](#appendix-b-the-physical-pendulum-case-study)

---

## 1. Overview

### What PanPhy Labs expects

PanPhy Labs is a **static PWA** deployed on GitHub Pages with:

- **No build system** — no Vite, Webpack, npm, TypeScript compiler
- **No frameworks** — no React, Vue, Svelte, Angular
- **Vanilla JavaScript only** — plain JS, HTML5, CSS3
- **Single-file simulations** — each simulation is one `.html` file (except complex ones like Collision Lab which use a modular structure)
- **Offline-first** — every published page works offline via a Service Worker
- **Consistent design** — shared CSS variable theming, banner header, footer, responsive layout

### What your Cloud Run simulation typically has

A typical AI Studio / Cloud Run simulation contains:

- A **React** (or similar) component tree in `src/App.tsx`
- **Tailwind CSS** utility classes for styling
- **Vite** (or similar) build tooling with hot module replacement
- **Docker** + **nginx** for Cloud Run deployment
- **Environment variable injection** (`runtime-config.js`, `.env`)
- A `package.json` with many unused dependencies (Gemini API, Express, SQLite, etc.)
- `metadata.json` for AI Studio

### The adaptation goal

Strip away everything except the core simulation logic (physics, canvas drawing, UI interactions) and repackage it as a vanilla HTML file that matches PanPhy Labs conventions.

---

## 2. Before You Start — Understand Both Worlds

Read these files before beginning any adaptation:

| File | Purpose |
|------|---------|
| `CLAUDE.md` | Full AI assistant guide with all conventions, patterns, and rules |
| `README.md` | Project philosophy and overview |
| `sw.js` | Service Worker — you will edit this |
| `index.html` | Landing page — you will add a card here |
| `sitemap.xml` | Sitemap — you will add the new URL |
| Any existing simulation | Reference for the target structure and style |

The most useful reference simulations to study (in order of complexity):

1. **`simulations/superposition.html`** — Simplest. Canvas-based, single-file, no external deps.
2. **`simulations/standing_wave.html`** — Similar but with more controls.
3. **`simulations/states.html`** — Uses Three.js (external CDN dependency).
4. **`simulations/collision.html`** — Modular (separate JS/CSS files + MediaPipe assets).

---

## 3. Step 1: Audit the Source Simulation

Before writing any code, thoroughly analyse the source repository.

### 3a. Identify the core simulation logic

Open the main component file (usually `src/App.tsx` or `src/App.jsx`). Answer:

- **What physics does it simulate?** (equations, constants, parameters)
- **What rendering method does it use?** (Canvas 2D, WebGL/Three.js, SVG, DOM animation)
- **What user interactions exist?** (sliders, drag, click, keyboard)
- **What UI state is there?** (play/pause, toggle visibility, parameter values)

The core logic is almost always **entirely in one file** — the React component. This is the only code you need to port.

### 3b. Identify what to discard

Everything related to Cloud Run / AI Studio infrastructure must go:

| File/Code | Why it exists | Action |
|-----------|--------------|--------|
| `Dockerfile` | Cloud Run container build | Delete / ignore |
| `docker-entrypoint.sh` | Runtime env injection | Delete / ignore |
| `nginx.conf` | Cloud Run web server | Delete / ignore |
| `public/runtime-config.js` | API key injection | Delete / ignore |
| `.env` / `.env.example` | Environment variables | Delete / ignore |
| `metadata.json` | AI Studio metadata | Delete / ignore |
| `vite.config.ts` | Vite build config | Delete / ignore |
| `tsconfig.json` | TypeScript config | Delete / ignore |
| `package.json` | NPM dependencies | Delete / ignore |
| `package-lock.json` | NPM lockfile | Delete / ignore |

### 3c. Identify unused dependencies

Many AI Studio projects ship dependencies that are **never imported in the actual simulation code**. Check `package.json` and cross-reference with actual `import` statements in `src/`. Common unused deps:

- `@google/genai` — Gemini API (often scaffolded but never called)
- `express` — Backend server (not needed for client-side sim)
- `better-sqlite3` — Database (not needed)
- `dotenv` — Env loading (not needed)
- `motion` / `framer-motion` — Animation library (often unused)

### 3d. Identify actual external dependencies

If the simulation actually **uses** an external library (Three.js, Chart.js, Plotly, etc.), you'll need to load it from a CDN in the adapted version. Note the library and version.

---

## 4. Step 2: Extract the Core Simulation Logic

### 4a. Physics and constants

Copy out all physics constants and equations. For example, from the Physical Pendulum:

```javascript
const L = 1.0;       // Rod length (m)
const g = 9.81;      // Gravity (m/s^2)
const theta0 = 0.25; // Initial angle (rad)

// Parallel axis theorem: I = I_cm + md^2
const I = (1/12) * L * L + d * d;

// Angular frequency
const omega = Math.sqrt((g * d) / I);

// Period
const T = 2 * Math.PI * Math.sqrt(I / (m * g * d));
```

### 4b. Canvas drawing code

The canvas drawing logic from a React `useEffect` translates almost directly to vanilla JS. The key changes:

| React pattern | Vanilla JS equivalent |
|--------------|----------------------|
| `const canvasRef = useRef()` | `document.getElementById('myCanvas')` |
| `useEffect(() => { ... }, [])` | Just call the function directly |
| `requestRef.current = requestAnimationFrame(render)` | `requestAnimationFrame(render)` (store in a plain variable) |
| Reading state via `someRef.current` | Read the variable directly (no ref indirection needed) |

### 4c. User interactions

Convert React event handlers to vanilla DOM event listeners:

| React | Vanilla JS |
|-------|-----------|
| `onPointerDown={handlePointerDown}` | `canvas.addEventListener('pointerdown', handlePointerDown)` |
| `onChange={(e) => setValue(e.target.value)}` | `slider.addEventListener('input', function() { ... })` |
| `onClick={() => setPlaying(!playing)}` | `btn.addEventListener('click', function() { ... })` |
| `useState(initialValue)` | Plain `let` variable |

### 4d. Tailwind to CSS

Tailwind utility classes must be converted to equivalent CSS. Some common translations:

| Tailwind | CSS |
|----------|-----|
| `bg-stone-50` | `background: #fafaf9` (but use CSS variables instead) |
| `p-4` | `padding: 16px` |
| `rounded-2xl` | `border-radius: 16px` |
| `text-2xl font-black` | `font-size: 1.5rem; font-weight: 900` |
| `grid grid-cols-2 gap-4` | `display: grid; grid-template-columns: 1fr 1fr; gap: 16px` |
| `flex items-center justify-between` | `display: flex; align-items: center; justify-content: space-between` |
| `space-y-6` | Child margins: `> * + * { margin-top: 1.5rem }` or use `gap` with flex/grid |

**Important**: Don't use Tailwind's exact colours. Map them to PanPhy Labs CSS variables instead. See [Appendix A](#appendix-a-css-variable-reference).

### 4e. Lucide/icon libraries

If the source uses icon libraries (Lucide, Heroicons, etc.), replace them with inline SVGs. You only need 3-5 icons for a typical simulation (play, pause, reset, settings gear, info circle). These are small SVGs that can be inlined directly in the HTML.

---

## 5. Step 3: Rewrite as a Single-File Vanilla HTML Page

Use this skeleton as your starting point:

```html
<!DOCTYPE html>
<html lang="en" data-theme="light">

<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover" />
    <meta name="theme-color" content="#f8f9fa">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="default">

    <!-- FOUC-free dark theme detection -->
    <script>
    (function() {
      if (localStorage.getItem('YOUR-SIM-NAME-dark')) {
        document.documentElement.setAttribute('data-theme', 'dark');
        var tc = document.querySelector('meta[name="theme-color"]');
        var sb = document.querySelector('meta[name="apple-mobile-web-app-status-bar-style"]');
        if (tc) tc.setAttribute('content', '#0f1014');
        if (sb) sb.setAttribute('content', 'black-translucent');
      }
    })();
    </script>

    <title>Your Simulation Title</title>
    <link rel="icon" type="image/png" href="/assets/favicon.png">
    <script src="/assets/sw-register.js" defer></script>

    <style>
        /* Full CSS here — see Section 6 for the design system */
    </style>
</head>

<body>
    <!-- Banner header -->
    <header class="banner">
        <div class="banner-logo">
            <a href="https://panphy.github.io/">
                <img src="/assets/panphy.png" alt="PanPhy logo">
            </a>
        </div>
        <div class="banner-title">Your Simulation Title</div>
        <div class="banner-actions">
            <div class="theme-toggle">
                <button id="theme-button" aria-label="Toggle Light/Dark Theme">&#9728;&#65039;</button>
            </div>
        </div>
    </header>

    <!-- Main content -->
    <div class="main-content">
        <!-- Canvas, controls, educational content... -->
    </div>

    <!-- Footer -->
    <footer>
        &copy; <a href="/">PanPhy Labs</a> |
        <a href="https://buymeacoffee.com/panphy" target="_blank" rel="noopener noreferrer">Support My Projects</a>
    </footer>

    <!-- JavaScript -->
    <script>
        // Theme toggle boilerplate (see below)
        // Physics simulation logic
        // Event listeners
    </script>

    <!-- Brave iOS gradient text fix -->
    <script>
        (function() {
            if (/iP(hone|ad|od)/.test(navigator.userAgent) && navigator.brave) return;
            document.querySelectorAll('.banner-title').forEach(function(el) { el.classList.add('gradient-text'); });
        })();
    </script>
</body>

</html>
```

### Theme toggle boilerplate

Every simulation needs this exact pattern for theme toggling:

```javascript
const themeButton = document.getElementById('theme-button');
const DARK_THEME_KEY = 'your-sim-name-dark';
const LIGHT_THEME_KEY = 'your-sim-name-light';

const currentTheme = localStorage.getItem(DARK_THEME_KEY)
    ? 'dark'
    : localStorage.getItem(LIGHT_THEME_KEY)
        ? 'light'
        : 'light';

document.documentElement.setAttribute('data-theme', currentTheme);
updateAppChromeTheme(currentTheme);
updateButtonEmoji(currentTheme);

function updateAppChromeTheme(theme) {
    const themeColorMeta = document.querySelector('meta[name="theme-color"]');
    const appleStatusBarMeta = document.querySelector('meta[name="apple-mobile-web-app-status-bar-style"]');
    const isDark = theme === 'dark';
    if (themeColorMeta) themeColorMeta.setAttribute('content', isDark ? '#0f1014' : '#f8f9fa');
    if (appleStatusBarMeta) appleStatusBarMeta.setAttribute('content', isDark ? 'black-translucent' : 'default');
}

window.addEventListener('pageshow', function() {
    updateAppChromeTheme(document.documentElement.getAttribute('data-theme'));
});

function updateButtonEmoji(theme) {
    themeButton.textContent = theme === 'dark' ? '\u{1F319}' : '\u{2600}\u{FE0F}';
}

themeButton.addEventListener('click', function() {
    let theme = document.documentElement.getAttribute('data-theme');
    if (theme === 'light') {
        document.documentElement.setAttribute('data-theme', 'dark');
        localStorage.setItem(DARK_THEME_KEY, 'true');
        localStorage.removeItem(LIGHT_THEME_KEY);
        updateAppChromeTheme('dark');
        updateButtonEmoji('dark');
    } else {
        document.documentElement.setAttribute('data-theme', 'light');
        localStorage.setItem(LIGHT_THEME_KEY, 'true');
        localStorage.removeItem(DARK_THEME_KEY);
        updateAppChromeTheme('light');
        updateButtonEmoji('light');
    }
});

// Offline font fallback
function updateOfflineFontState() {
    document.documentElement.classList.toggle('offline-font', !navigator.onLine);
}
window.addEventListener('online', updateOfflineFontState);
window.addEventListener('offline', updateOfflineFontState);
updateOfflineFontState();
```

### Canvas animation pattern

```javascript
let previousTime = null;
let isPlaying = true;

function render(time) {
    if (previousTime !== null) {
        const deltaTime = Math.min((time - previousTime) / 1000, 0.1);
        // Physics update using deltaTime
        // Canvas drawing
    }
    previousTime = time;

    if (isPlaying) {
        requestAnimationFrame(render);
    }
}

requestAnimationFrame(render);
```

### Canvas resize pattern

```javascript
function resizeCanvas() {
    const aspectRatio = 4 / 3; // Choose appropriate ratio
    const width = canvas.clientWidth;
    const height = width / aspectRatio;
    const dpr = window.devicePixelRatio || 1;
    canvas.style.height = height + 'px';
    canvas.width = Math.round(width * dpr);
    canvas.height = Math.round(height * dpr);
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
}

resizeCanvas();
window.addEventListener('resize', resizeCanvas);
```

---

## 6. Step 4: Apply the PanPhy Labs Design System

### 6a. CSS variables (mandatory)

All colours must use CSS custom properties. Never hardcode colour values. The simulation must support both light and dark themes.

Here is the core set of variables every simulation should define:

```css
:root {
    /* Fonts */
    --font-sans: 'Inter', sans-serif;
    --font-display: 'Poppins', 'Inter', sans-serif;
    --font-mono: 'JetBrains Mono', monospace;

    /* Backgrounds */
    --bg-color: #f8f9fa;
    --bg-pattern: #e9ecef;

    /* Text */
    --text-main: #2d3436;
    --text-secondary: #636e72;

    /* Brand */
    --brand-primary: #6c5ce7;
    --brand-secondary: #a29bfe;
    --brand-accent: #00cec9;

    /* Cards */
    --card-bg: #ffffff;
    --card-border: #e9ecef;
    --card-shadow: 0 10px 30px rgba(0, 0, 0, 0.05);

    /* Canvas */
    --canvas-bg: #ffffff;

    /* Sliders */
    --slider-track: #e2e8f0;
    --slider-thumb: #6c5ce7;

    /* Navigation */
    --nav-bg: rgba(255, 255, 255, 0.85);
    --nav-border: rgba(255, 255, 255, 0.5);
}

[data-theme="dark"] {
    --bg-color: #0f1014;
    --bg-pattern: #181a20;
    --text-main: #dfe6e9;
    --text-secondary: #b2bec3;
    --brand-primary: #a29bfe;
    --brand-secondary: #6c5ce7;
    --brand-accent: #81ecec;
    --card-bg: #1e2129;
    --card-border: #2d3436;
    --card-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
    --canvas-bg: #16181d;
    --slider-track: #4a5568;
    --slider-thumb: #a29bfe;
    --nav-bg: rgba(22, 24, 29, 0.85);
    --nav-border: rgba(255, 255, 255, 0.08);
}
```

Add simulation-specific variables as needed (e.g., `--rod-fill`, `--pivot-color`).

### 6b. Canvas drawing with theme awareness

When drawing on canvas, read CSS variables at draw time:

```javascript
function getCSSVar(name) {
    return getComputedStyle(document.documentElement).getPropertyValue(name).trim();
}

// Use in drawing:
ctx.fillStyle = getCSSVar('--rod-fill');
ctx.strokeStyle = getCSSVar('--brand-primary');
```

This ensures canvas colours update when the theme toggles.

### 6c. Responsive design rules

- Use `max-width: 1200px` for main content
- Grid layout: single column on mobile, split on desktop (`@media (min-width: 960px)`)
- Touch targets: 44px minimum (`@media (pointer: coarse)`)
- Respect reduced motion: `@media (prefers-reduced-motion: reduce)`
- Use `viewport-fit=cover` for notched devices
- Font import: `@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&family=JetBrains+Mono:wght@400;600&family=Poppins:wght@600;700;800&display=swap')`

### 6d. Body background

```css
body {
    background-color: var(--bg-color);
    background-image: radial-gradient(var(--bg-pattern) 1px, transparent 1px);
    background-size: 30px 30px;
}
```

### 6e. Banner header (copy exactly)

The banner header is identical across all simulations. Copy it from any existing sim and change only the title text.

### 6f. Footer (copy exactly)

```html
<footer>
    &copy; <a href="/">PanPhy Labs</a> |
    <a href="https://buymeacoffee.com/panphy" target="_blank" rel="noopener noreferrer">Support My Projects</a>
</footer>
```

---

## 7. Step 5: Integrate with the Site Infrastructure

Once the simulation HTML file is complete, you need to register it with the site. There are **five** files to update:

### 7a. Place the file

Put the HTML file in the correct directory:

```
simulations/your_simulation.html
```

Use snake_case for filenames.

### 7b. Add a tile card to `index.html`

Find the Simulations section (`<div class="section" id="simulations">`) and add a card:

```html
<div class="card">
    <h3>Your Simulation Title</h3>
    <p>One-sentence description of what the simulation does.</p>
    <a href="/simulations/your_simulation.html">Launch Sim</a>
</div>
```

### 7c. Add to `OFFLINE_CARD_REQUIREMENTS` in `index.html`

Find the `OFFLINE_CARD_REQUIREMENTS` object in the `<script>` section at the bottom of `index.html`. Add an entry:

```javascript
'/simulations/your_simulation.html': ['/simulations/your_simulation.html'],
```

If your simulation uses external CDN libraries, list them too:

```javascript
'/simulations/your_simulation.html': [
    '/simulations/your_simulation.html',
    'https://cdn.jsdelivr.net/npm/three@0.161.0/build/three.module.js'
],
```

### 7d. Add to `ASSETS_TO_CACHE` in `sw.js`

Open `sw.js` and add the page path to the `ASSETS_TO_CACHE` array in the Simulations section:

```javascript
// Simulations
'/simulations/your_simulation.html',
```

If the simulation has external CDN dependencies, add those URLs too (they must match exactly what's in the HTML `<script src="...">`).

**Then bump the `BUILD_ID`:**

```javascript
const BUILD_ID = '2026-03-15T22:30:00Z';  // Update to current timestamp
```

The `BUILD_ID` format is ISO 8601: `YYYY-MM-DDTHH:MM:SSZ`. Always update this when you change any cached asset.

### 7e. Add to `sitemap.xml`

```xml
<url>
    <loc>https://panphy.github.io/simulations/your_simulation.html</loc>
</url>
```

---

## 8. Step 6: Verify Everything Works

### Local testing

```bash
cd /path/to/apps4you
python3 -m http.server 8000
```

Then check:

1. **Page loads** — `http://localhost:8000/simulations/your_simulation.html`
2. **Theme toggle** — Light/dark switch works, canvas colours update
3. **Simulation runs** — Animation plays, physics looks correct
4. **Controls work** — Sliders, buttons, toggles, drag interactions
5. **Responsive** — Resize browser window, check mobile layout
6. **Landing page** — Card appears in Simulations section at `http://localhost:8000/`
7. **Offline** — After first load, go offline (DevTools > Network > Offline) and reload

### Common issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Page doesn't load offline | Missing from `ASSETS_TO_CACHE` or `BUILD_ID` not bumped | Add to `sw.js` and bump `BUILD_ID` |
| Canvas is blank in dark mode | Hardcoded colours instead of CSS variables | Use `getCSSVar()` for all canvas colours |
| Slider doesn't update canvas | Event listener missing or wrong | Use `'input'` event, not `'change'` |
| Layout breaks on mobile | Missing responsive breakpoints | Add `@media (max-width: 960px)` rules |
| Touch drag doesn't work | Missing pointer capture | Call `canvas.setPointerCapture(e.pointerId)` on pointerdown |

---

## 9. Quick Reference Checklist

Use this checklist for every simulation you adapt:

### Source Analysis
- [ ] Identified core physics logic (constants, equations, update loop)
- [ ] Identified rendering method (Canvas 2D / Three.js / SVG)
- [ ] Identified user interactions (sliders, drag, keyboard)
- [ ] Identified external library dependencies (if any)
- [ ] Confirmed which dependencies are actually unused

### Rewrite
- [ ] Created single `.html` file in `simulations/`
- [ ] Converted React state to plain variables
- [ ] Converted React effects to direct function calls
- [ ] Converted JSX event handlers to `addEventListener`
- [ ] Converted Tailwind classes to CSS with variables
- [ ] Replaced icon library with inline SVGs
- [ ] Removed all TypeScript (converted to plain JS)
- [ ] Removed all Cloud Run / AI Studio infrastructure

### Design System
- [ ] CSS variables for all colours (light + dark)
- [ ] Banner header with PanPhy logo and theme toggle
- [ ] Footer with copyright and support link
- [ ] Responsive layout (mobile + desktop)
- [ ] Touch-friendly controls (44px+ targets)
- [ ] `prefers-reduced-motion` respected
- [ ] Offline font fallback
- [ ] Brave iOS gradient text fix

### Site Integration
- [ ] File placed in correct directory
- [ ] Card added to `index.html` Simulations section
- [ ] Added to `OFFLINE_CARD_REQUIREMENTS` in `index.html`
- [ ] Added to `ASSETS_TO_CACHE` in `sw.js`
- [ ] `BUILD_ID` bumped in `sw.js`
- [ ] URL added to `sitemap.xml`

### Testing
- [ ] Simulation loads and runs correctly
- [ ] Theme toggle works (both light and dark)
- [ ] All controls function properly
- [ ] Responsive layout at all breakpoints
- [ ] Works offline after first visit
- [ ] Card appears on landing page
- [ ] Offline pill shows "Offline ready" on landing page

---

## 10. Common Adaptation Patterns

### Pattern: React `useState` → Plain variable

```jsx
// React
const [value, setValue] = useState(0.5);
```

```javascript
// Vanilla
let value = 0.5;
```

### Pattern: React `useRef` → Plain variable

```jsx
// React
const phaseRef = useRef(0);
// ... phaseRef.current += delta;
```

```javascript
// Vanilla
let phase = 0;
// ... phase += delta;
```

### Pattern: React `useEffect` (animation loop) → Direct call

```jsx
// React
useEffect(() => {
    const render = (time) => { /* ... */ requestAnimationFrame(render); };
    requestAnimationFrame(render);
    return () => cancelAnimationFrame(id);
}, []);
```

```javascript
// Vanilla
function render(time) { /* ... */ requestAnimationFrame(render); }
requestAnimationFrame(render);
```

### Pattern: React conditional rendering → DOM visibility

```jsx
// React
{showPanel && <div className="panel">...</div>}
```

```javascript
// Vanilla
panel.style.display = showPanel ? '' : 'none';
```

### Pattern: Three.js import (if simulation uses 3D)

```jsx
// React + Vite
import * as THREE from 'three';
```

```html
<!-- Vanilla: use importmap or CDN script -->
<script type="importmap">
{
    "imports": {
        "three": "https://cdn.jsdelivr.net/npm/three@0.161.0/build/three.module.js"
    }
}
</script>
<script type="module">
import * as THREE from 'three';
// ...
</script>
```

### Pattern: Tailwind responsive → CSS media queries

```jsx
// Tailwind
<div className="grid grid-cols-1 lg:grid-cols-12 gap-8">
    <div className="lg:col-span-7">...</div>
    <div className="lg:col-span-5">...</div>
</div>
```

```css
/* CSS */
.main-content {
    display: grid;
    grid-template-columns: 1fr;
    gap: 20px;
}

@media (min-width: 960px) {
    .main-content {
        grid-template-columns: 7fr 5fr;
    }
}
```

### Pattern: Hardcoded Stone/Indigo colours → CSS variables

| Tailwind colour | PanPhy Labs variable |
|----------------|---------------------|
| `stone-50` / `stone-100` | `--readout-bg` or `--bg-color` |
| `stone-200` | `--card-border` or `--rod-fill` |
| `stone-400` | `--text-secondary` |
| `stone-800` / `stone-900` | `--text-main` |
| `indigo-600` | `--brand-primary` |
| `indigo-700` | `--section-heading` |
| `emerald-500` | `--brand-accent` (or a sim-specific variable) |
| `amber-50` / `amber-200` | `--info-bg` / `--info-border` |

---

## Appendix A: CSS Variable Reference

### Core variables (used by all simulations)

| Variable | Light | Dark | Usage |
|----------|-------|------|-------|
| `--bg-color` | `#f8f9fa` | `#0f1014` | Page background |
| `--bg-pattern` | `#e9ecef` | `#181a20` | Dot pattern overlay |
| `--text-main` | `#2d3436` | `#dfe6e9` | Primary text |
| `--text-secondary` | `#636e72` | `#b2bec3` | Secondary/muted text |
| `--brand-primary` | `#6c5ce7` | `#a29bfe` | Primary accent (indigo) |
| `--brand-secondary` | `#a29bfe` | `#6c5ce7` | Secondary accent |
| `--brand-accent` | `#00cec9` | `#81ecec` | Teal accent |
| `--card-bg` | `#ffffff` | `#1e2129` | Card backgrounds |
| `--card-border` | `#e9ecef` | `#2d3436` | Card borders |
| `--card-shadow` | light shadow | dark shadow | Card shadows |
| `--canvas-bg` | `#ffffff` | `#16181d` | Canvas background |
| `--slider-track` | `#e2e8f0` | `#4a5568` | Slider track |
| `--slider-thumb` | `#6c5ce7` | `#a29bfe` | Slider thumb |
| `--nav-bg` | `rgba(255,255,255,0.85)` | `rgba(22,24,29,0.85)` | Header background |
| `--nav-border` | `rgba(255,255,255,0.5)` | `rgba(255,255,255,0.08)` | Header border |

### Simulation-specific variables (add as needed)

Define these for your simulation's unique visual elements. Example from Physical Pendulum:

| Variable | Light | Dark | Usage |
|----------|-------|------|-------|
| `--rod-fill` | `#e7e5e4` | `#3d3d3d` | Rod body fill |
| `--rod-stroke` | `#a8a29e` | `#6b6b6b` | Rod outline |
| `--pivot-color` | `#6c5ce7` | `#a29bfe` | Pivot marker |
| `--simple-bob` | `#00cec9` | `#81ecec` | Equivalent pendulum bob |
| `--grid-line` | `rgba(0,0,0,0.04)` | `rgba(255,255,255,0.04)` | Canvas grid |

---

## Appendix B: The Physical Pendulum Case Study

### Source: `physical-pendulum` repository

- **Framework**: React 19 + TypeScript
- **Styling**: Tailwind CSS v4
- **Build**: Vite
- **Deployment**: Google Cloud Run (Docker + nginx)
- **Dependencies**: 8 npm packages (only 2 actually used: `react`, `react-dom`)
- **Core logic**: 500 lines in `src/App.tsx`

### What was discarded

| Item | Lines/Files | Reason |
|------|------------|--------|
| `Dockerfile` | 15 lines | Cloud Run only |
| `docker-entrypoint.sh` | 15 lines | Runtime env injection |
| `nginx.conf` | 30 lines | Cloud Run web server |
| `public/runtime-config.js` | 5 lines | API key placeholder |
| `.env.example` | 3 lines | Environment variables |
| `metadata.json` | 5 lines | AI Studio metadata |
| `vite.config.ts` | 25 lines | Vite build config |
| `tsconfig.json` | 30 lines | TypeScript config |
| `package.json` + lockfile | 2 files | NPM ecosystem |
| `src/main.tsx` | 10 lines | React DOM mount |
| `src/index.css` | 3 lines | Tailwind import |
| React framework | ~100 lines overhead | useState, useEffect, useRef, JSX |
| Tailwind classes | ~200 class names | Replaced with CSS variables |
| Lucide icons | 6 imports | Replaced with inline SVGs |
| TypeScript types | ~30 type annotations | Removed (plain JS) |

### What was kept

| Item | Lines | Notes |
|------|-------|-------|
| Physics constants | 4 | `L`, `g`, `theta0`, `m` |
| Physics calculations | 8 | Parallel axis theorem, omega, period |
| Canvas drawing (rod) | 25 | Nearly unchanged |
| Canvas drawing (CM marker) | 20 | Unchanged |
| Canvas drawing (simple pendulum) | 30 | Nearly unchanged |
| Drag interaction logic | 20 | Converted from React events |
| Educational text content | 80 lines HTML | Restructured from JSX |
| Readout calculations | 6 | Unchanged |

### What was added

| Item | Lines | Notes |
|------|-------|-------|
| CSS with variables (light + dark) | ~350 | Replaces Tailwind |
| Theme toggle boilerplate | 40 | Standard PanPhy pattern |
| Offline font fallback | 6 | Standard pattern |
| Banner header HTML | 12 | Standard pattern |
| Footer HTML | 4 | Standard pattern |
| Canvas resize logic | 12 | DPR-aware responsive |
| `getCSSVar()` helper | 3 | For theme-aware canvas drawing |
| Brave iOS gradient fix | 5 | Standard pattern |

### Result

- **Source**: 500 lines of React/TSX + 8 config files + Docker infrastructure
- **Output**: One self-contained HTML file (~750 lines including CSS + educational content)
- **External dependencies**: Zero (pure vanilla JS + Canvas 2D)
- **Offline**: Works fully offline after first visit
- **Theme**: Full light/dark support matching site design system
