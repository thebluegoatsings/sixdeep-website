# Six Deep Studios LLC

The Six Deep Studios website: a static Astro site for the studio, its first project
**Big Time**, and a playable in-browser physics tech demo.

## Pages

| Route      | File                      | What it is                                              |
| ---------- | ------------------------- | ------------------------------------------------------- |
| `/`        | `src/pages/index.astro`   | Home: hero, live demo, Big Time, team, Steam status      |
| `/big-time`| `src/pages/big-time.astro`| Game feature breakdown and status                        |
| `/play-3d` | `src/pages/play-3d.astro` | Standalone full-width version of the wrecking-ball demo  |

## Local development

```bash
npm install
npm run dev
```

The site runs by default at http://localhost:4321.

Note: this dev server does not always pick up file edits reliably. If a change doesn't
appear, stop the server and start it again (`astro dev stop` then `astro dev --background`).

## Production build

```bash
npm run build
npm run preview
```

## GitHub Pages deployment

`.github/workflows/deploy.yml` builds the static site and deploys it to GitHub Pages on
pushes to `main`. `astro.config.mjs` sets `site` to `https://sixdeepstudios.com` — a
`CNAME` file still needs to be added to `public/` for that custom domain to stick.

## Project structure

```text
/
├── public/
│   ├── favicon.ico / favicon.svg
│   ├── logo.svg                      # Header brand mark
│   ├── og-image.jpg                  # 1200x630 social preview card
│   └── under-construction-crane.svg  # Artwork in the "Steam is coming" panel
├── src/
│   ├── assets/
│   │   └── sixdeepbanner.jpg         # Hero art; optimized at build via astro:assets
│   ├── components/
│   │   ├── SiteHeader.astro          # Brand lockup + nav (single source of nav links)
│   │   ├── SiteFooter.astro
│   │   ├── Icon.astro                # Inline SVG icon set
│   │   └── CraneGame3D.astro         # The Box3D wrecking-ball demo
│   ├── layouts/
│   │   └── Layout.astro              # <head>, meta tags, scroll-reveal script
│   ├── pages/
│   │   ├── index.astro
│   │   ├── big-time.astro
│   │   └── play-3d.astro
│   └── styles/
│       └── global.css                # Tokens, layout primitives, buttons, a11y
├── AGENTS.md
├── GAME-NOTES.md                     # Design notes for the demo
├── astro.config.mjs
└── package.json
```

## Conventions

- **Any style used by more than one page lives in `src/styles/global.css`.** Page- and
  component-specific styles stay in that file's own `<style>` block. If you're about to
  paste a rule into a second page, it belongs in `global.css` instead.
- **Navigation lives in `SiteHeader.astro` only.** Add a link there once and every page
  gets it; pass `current` to mark the active page.
- **Design tokens are CSS custom properties** in `global.css`: `--accent` (yellow),
  `--secondary` (blue), `--line`, `--panel`, plus shared `--radius-*`, `--shadow-*` and
  `--gutter` values. Change them there rather than hard-coding colours.
- **Images in `src/assets/`** are optimized by `astro:assets`. Images in `public/` are
  served as-is, so only put files there that need a fixed URL (favicon, OG card).

