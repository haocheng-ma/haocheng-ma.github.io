# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Haocheng Ma's academic website (https://mahaocheng.me), built on the [al-folio](https://github.com/alshedivat/al-folio) Jekyll theme. Deployed to GitHub Pages from the `gh-pages` branch; source lives on `master`.

## Common commands

Local development via Docker (preferred — matches CI):

```bash
docker compose up          # serve at http://localhost:8080 with livereload (port 35729)
docker compose -f docker-compose-slim.yml up   # slim image variant
```

Local development via Ruby/Bundler:

```bash
bundle install
bundle exec jekyll serve --livereload    # or: bin/cibuild for one-shot build
```

Formatting (Prettier + Liquid plugin, enforced by CI in `.github/workflows/prettier*.yml`):

```bash
npx prettier --check .
npx prettier --write .
```

Deploy to GitHub Pages (builds, purges CSS, force-pushes `gh-pages`):

```bash
bin/deploy                 # refuses to run if working tree is dirty
bin/deploy --no-push       # dry run
```

## Architecture notes

- **Jekyll + Liquid, not a SPA.** Pages are Markdown/Liquid files under `_pages/`, `_posts/`, and top-level `.md`. Layouts in `_layouts/*.liquid` wrap content; partials live in `_includes/`. Styling is Sass under `_sass/` and `assets/css/`.
- **`_config.yml` is the site's control plane.** Nearly all site-wide behavior (theme, navbar, social links, scholar/bib config, plugin toggles, jekyll-archives, pagination) is configured there. Changes to it require restarting Jekyll — `bin/entry_point.sh` watches the file and auto-restarts the server in Docker.
- **Custom Ruby plugins** in `_plugins/` extend Jekyll at build time. Notable ones:
  - `google-scholar-citations.rb` / `inspirehep-citations.rb` — fetch citation counts.
  - `download-3rd-party.rb` — vendors external JS/CSS so the built site is self-contained.
  - `external-posts.rb` — pulls in posts from external feeds.
  - `hide-custom-bibtex.rb`, `details.rb`, `remove-accents.rb`, `file-exists.rb`, `cache-bust.rb` — smaller Liquid filters/generators.
- **Publications** come from `_bibliography/papers.bib` (BibTeX) via `jekyll-scholar`, rendered by `_layouts/bib.liquid`. Add papers to the `.bib` file, not as separate pages.
- **Structured data** in `_data/` drives templated sections: `cv.yml` (CV page via `_layouts/cv.liquid`), `coauthors.yml`, `repositories.yml`, `venues.yml`.
- **Blog posts** are `_posts/YYYY-MM-DD-slug.md`. Long-form posts use `distill.liquid` layout for the Distill-style academic format; standard ones use `post.liquid`.
- **Deployment flow** (`bin/deploy`): builds with `JEKYLL_ENV=production`, runs `purgecss` against `purgecss.config.js` to strip unused CSS, then replaces the branch contents with `_site/` and force-pushes to `gh-pages`. The `master` → `gh-pages` build can also run via `.github/workflows/deploy.yml`.
- **CI** covers Prettier formatting, broken-link checks, axe accessibility, Lighthouse, and Docker image builds — see `.github/workflows/`.

## Gotchas

- `Gemfile.lock` behavior differs inside Docker: `bin/entry_point.sh` deletes it if untracked, restores it if tracked. Don't be surprised if it regenerates on container start.
- `bin/deploy` aborts on any uncommitted or untracked files — commit or stash first.
- The site URL is `https://mahaocheng.me` (set in `_config.yml`), not the default `github.io` URL.
