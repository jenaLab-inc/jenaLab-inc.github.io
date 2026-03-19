# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Jekyll-based technical blog for JenaLab, deployed on GitHub Pages. Uses the "Mediator" theme (Medium-inspired design).

## Build & Development Commands

```bash
# Install dependencies
bundle install

# Run local dev server (serves at http://localhost:4000)
bundle exec jekyll serve

# Build site without serving
bundle exec jekyll build
```

Output goes to `_site/` (gitignored). Deployment is automatic via GitHub Pages on push to `master`.

## Architecture

- **_config.yml** — Site configuration (title, author, pagination, plugins: jemoji + jekyll-paginate)
- **_layouts/** — `default.html` (base) → `post.html` (articles) and `page.html` (static pages)
- **_includes/** — Reusable partials: `head.html` (SEO/meta), `header.html`, `footer.html`, `javascripts.html`, `google_analytics.html`
- **_posts/** — Markdown articles with YAML front matter, named `YYYY-MM-DD-slug.markdown`
- **_sass/** — SASS sources including Bourbon mixin library
- **css/main.sass** — Primary stylesheet, compiles all SASS; uses mobile-first responsive approach
- **assets/** — `images/` (site-wide), `article_images/` (per-post, in folders matching post date/slug), `js/` (jQuery, fitvids, readingTime)

## Post Front Matter

Posts require YAML front matter with these fields:

```yaml
---
layout: post
title: "Post Title"
date: YYYY-MM-DD HH:MM:SS
categories: category-name
author: author_name
image: /assets/article_images/YYYY-MM-DD-slug/image.JPG      # desktop header
image2: /assets/article_images/YYYY-MM-DD-slug/image-mobile.JPG  # mobile header
---
```

## Key Integrations

- **Comments:** Giscus (GitHub Discussions) configured in `_layouts/post.html` — repo: `jenalab-inc/jenaLab-inc.github.io`, Korean language
- **Fonts:** Linux Libertine (serif body), Open Sans (sans-serif headings), Font Awesome 4.2.0 icons — loaded in `_includes/head.html`
- **JS features:** Reading time calculation, responsive video embeds (fitvids), parallax header images, image caption generation from alt text — wired in `_includes/javascripts.html` and inline in `post.html`

## Conventions

- Commit messages are in Korean
- Blog content is primarily in Korean
- Article images go in `/assets/article_images/YYYY-MM-DD-slug/` directories
- Homepage (`index.html`) supports featured posts via `featured: true` front matter flag
- Pagination is set to 5 posts per page
