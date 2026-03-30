# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Personal website (wbreiler.com) built with [Astro](https://astro.build/) and the Smallworld theme (Pico CSS). Static site — no client-side JavaScript.

Remote: `https://git.wbreiler.com/wbreiler/wbreiler.com.git`

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
- **[src/content/blog/](src/content/blog/)** — Markdown blog posts with `title`, `description`, and `pubDate` frontmatter
- **[src/components/PageHeader.astro](src/components/PageHeader.astro)** — edit navigation links here
- **[src/layouts/](src/layouts/)** — `Base.astro` wraps all pages, `BlogPost.astro` wraps blog posts
- **[src/styles/](src/styles/)** — `main.scss` controls which Pico CSS modules are included

## Adding content

**New blog post** — create `src/content/blog/my-post.md`:

```markdown
---
title: Post title
description: Short description
pubDate: 2026-01-01
---

Post content here.
```

**New project** — edit [src/pages/projects.astro](src/pages/projects.astro) and add a `<section>` block.

**New page** — create `src/pages/my-page.astro` and add a `<NavLink>` entry in `PageHeader.astro`.
