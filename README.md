# imranemon.tech

> Blog + portfolio of Imran Emon — .NET & Cloud Engineer.
> Static site built with [Astro](https://astro.build), deployed to GitHub Pages at [imranemon.tech](https://imranemon.tech).

## Structure

- `src/pages/index.astro` — portfolio home (intro, skills, projects, recent posts)
- `src/pages/blog/` — blog index + post pages
- `src/content/blog/` — markdown posts (frontmatter: title, description, pubDate, tags, draft)
- `src/pages/rss.xml.js` — RSS feed
- `public/robots.txt` — welcomes search + AI crawlers
- `.github/workflows/deploy.yml` — builds and deploys to GitHub Pages on push to main

## Writing a post

```bash
# 1. create a draft
$EDITOR src/content/blog/my-post-slug.md
```

```markdown
---
title: My post title
description: One-line summary shown in lists and feeds.
pubDate: 2026-09-06
tags: [dotnet, kubernetes]
draft: true
---

Body in markdown...
```

Set `draft: false` when the post is ready to publish. `draft: true` posts are
excluded from the build entirely.

## Local development

```bash
npm install
npm run dev      # http://localhost:4321
npm run build    # outputs to dist/
```

## Deployment

Push to `main` — GitHub Actions builds and deploys. Custom domain
(imranemon.tech) is configured via the `CNAME` file + GitHub Pages settings.
