# Utkarsh Mehrotra - Personal Portfolio

A static personal portfolio and engineering blog for [Utkarsh Mehrotra](https://github.com/utkarsh-mehrotra).

## 🌍 Live Website

**[https://utkarsh-mehrotra.github.io/](https://utkarsh-mehrotra.github.io/)**

## 📁 Structure

| Path | Purpose |
|------|---------|
| `index.html` | Home page (light/dark theme, follows system preference with a manual toggle) |
| `wu-pipeline.html` | Article: 100M events/day fraud detection pipeline |
| `samsung-finance.html` | Article: multi-market lending platform correctness |
| `scaling-fraud-detection.html` | Article: thought experiment on scaling to 50K events/sec |
| `jpmc-scale.html`, `index-dark.html` | Redirect stubs for old URLs |
| `favicon.svg`, `robots.txt`, `sitemap.xml` | Site metadata |
| `notes/` | Private working notes (**not deployed**) |
| `assets/blogs/` | Article PDFs (**not deployed**, not linked) |

## 🛠️ Tech Stack

- Plain HTML5, vanilla CSS and vanilla JavaScript: no frameworks, no build step
- Google Fonts: Inter (home page), Inter + JetBrains Mono (articles)

## 🚀 Local Development

Open `index.html` in a browser. No install or build is required.

## 🤖 Deployment

`.github/workflows/deploy.yml` deploys `main` to GitHub Pages on every push. It copies only the public files (`*.html`, `favicon.svg`, `robots.txt`, `sitemap.xml`) into `_site/` and publishes that folder. **When you add a new public file type, add it to the "Assemble site" step.**
