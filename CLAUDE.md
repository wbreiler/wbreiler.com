# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Personal website (wbreiler.com) built with [Astro](https://astro.build/) and the Smallworld theme (Pico CSS). Static site — minimal client-side JavaScript (Giscus comments widget only).

Remotes: `https://git.wbreiler.com/wbreiler/wbreiler.com.git` (primary), `https://github.com/wbreiler/wbreiler.com` (GitHub, also used for Giscus comments via GitHub Discussions)

## Commands

```bash
npm run dev      # Start dev server (localhost:4321)
npm run build    # Build for production → dist/
npm run preview  # Preview production build locally
npm run format   # Format with Prettier
```

## Architecture

- **[src/settings.ts](src/settings.ts)** — site title, description, OG image config
- **[src/pages/](src/pages/)** — one `.astro` file per page; add new pages here
- **[src/content/blog/](src/content/blog/)** — Markdown blog posts (see frontmatter format below)
- **[src/components/PageHeader.astro](src/components/PageHeader.astro)** — navigation links
- **[src/components/PageFooter.astro](src/components/PageFooter.astro)** — footer links
- **[src/layouts/Base.astro](src/layouts/Base.astro)** — wraps all pages
- **[src/layouts/BlogPost.astro](src/layouts/BlogPost.astro)** — wraps blog posts; contains the Giscus comments script
- **[src/styles/](src/styles/)** — `main.scss` controls which Pico CSS modules are included
- **[src/templates/post.md](src/templates/post.md)** — reference template for new blog posts

## Adding content

**New blog post** — create `src/content/blog/my-post.md`. Tags are optional but preferred:

```markdown
---
title: Post title
description: Short description
pubDate: 2026-01-01
tags:
  - homelab
  - proxmox
---

Post content here.
```

**New project** — edit [src/pages/projects.astro](src/pages/projects.astro) and add a `<section>` block.

**New page** — create `src/pages/my-page.astro` and add a `<NavLink>` entry in [PageHeader.astro](src/components/PageHeader.astro).

## Comments (Giscus)

Comments on blog posts are powered by [Giscus](https://giscus.app), backed by GitHub Discussions on the `wbreiler/wbreiler.com` repo. The script is in `BlogPost.astro`. No changes needed unless the repo or category IDs change.

## Theming

Dark/light mode follows the visitor's OS preference automatically via Pico CSS — there is no manual toggle. The Giscus widget uses `preferred_color_scheme` to match.
