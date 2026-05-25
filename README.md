# README

Hugo theme and content for personal blog - [https://ibreak.software](https://ibreak.software)

## Workflow

### Create a new post

```bash
hugo new post/my-new-post-title.md
```

This scaffolds `content/post/my-new-post-title.md` from `archetypes/default.md`. Open it in your editor (VS Code / Zed / etc), fill in the body in Markdown, and update the frontmatter:

```yaml
---
title: "My New Post Title"
date: 2026-05-26
categories:
- some-category
tags:
- some-tag
thumbnailImagePosition: left
thumbnailImage: /img/my-new-post-title/cover.png
draft: true
---
```

Put images under `static/img/<post-slug>/`. Set `draft: false` when ready to publish.

### Edit an existing post

Files live in `content/post/*.md`. Open the relevant file in your editor and edit Markdown directly. No rename needed — the URL is derived from the filename via the permalink rule `post = "/:year/:month/:slug/"` in `config.toml`.

### Preview locally

```bash
hugo server -D
```

Open http://localhost:1313/. `-D` includes drafts. Live-reloads on save.

### Publish (deploy to Netlify)

Push to `master` — Netlify watches the GitHub repo and rebuilds automatically.

```bash
git add content/post/my-new-post-title.md static/img/my-new-post-title/
git commit -m "add post: my new post title"
git push origin master
```

Build status: Netlify dashboard.
