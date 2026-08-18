# Six Deep Studios LLC

A lightweight Astro landing page for Six Deep Studios LLC with a modern studio aesthetic, an under-construction crane-and-beer-cans placeholder visual, an About Us section, and a placeholder Steam link.

## Local development

```bash
npm install
npm run dev
```

The site runs by default at http://localhost:4321.

## Production build

```bash
npm run build
npm run preview
```

## GitHub Pages deployment

This project includes a GitHub Actions workflow that builds the static Astro site and deploys it to GitHub Pages automatically on pushes to the main branch.

## Project structure

```text
/
├── public/
│   ├── favicon.svg
│   └── under-construction-crane.svg
├── src/
│   ├── layouts/
│   │   └── Layout.astro
│   ├── pages/
│   │   └── index.astro
│   └── styles/
│       └── global.css
├── .github/
│   └── workflows/
│       └── deploy.yml
├── astro.config.mjs
├── package.json
└── README.md
```
