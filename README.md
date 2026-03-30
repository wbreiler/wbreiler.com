# wbreiler.com

Personal site and blog built with [Astro](https://astro.build/) and the [Smallworld](https://github.com/anaxite/astro-smallworld) theme (Pico CSS).

## Development

```bash
npm install
npm run dev      # localhost:4321
npm run build    # output → dist/
npm run preview  # preview production build
npm run format   # format with Prettier
```

## Adding a blog post

Create a Markdown file in `src/content/blog/`:

```markdown
---
title: Post title
description: Short description
pubDate: 2026-01-01
tags:
  - homelab
---

Post content here.
```

See `src/templates/post.md` for a full reference.
