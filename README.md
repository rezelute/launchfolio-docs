## What is Launchfolio?

Launchfolio is a self-hostable personal portfolio, blog, and CV app. The core idea is simple: **you bring the data, Launchfolio renders it**. Your content lives in a `persona-data` directory you control - YAML files, blog posts, and images. The app reads from it without you ever needing to touch application code.

```
persona-data/ ← your content (edit this freely)
launchfolio/ ← the app (leave alone)
```

Everything you need to customise the image is exposed through your `persona-data` directory and environment variables.

---

## Setting Up Your persona-data Directory

Before running the image, create a directory with this structure:

```
persona-data/
   basics.yaml        # your name, location, job title, social links
   site.yaml          # site name, strapline, footer text
   education.yaml     # degrees, certifications
   experience.yaml    # work history
   interests.yaml     # personal interests shown on the home page
   projects.yaml      # your portfolio projects
   cv-variants.yaml   # one or more tailored CV versions
   blogs/
      my-first-post.md
   img/
      site-logo.svg
      favicon/
         favicon.ico
         favicon.svg
         favicon-96x96.png
         apple-touch-icon.png
         web-app-manifest-192x192.png
         web-app-manifest-512x512.png
      blogs/
         my-post-cover.webp
      projects/
         my-project/
            cover.webp
            screenshot-1.webp
```

Each YAML file has a defined schema that is validated at server startup. If a file fails validation, the server will refuse to start and print the exact field errors to the console - so you'll know immediately what to fix.

Full JSON schemas and example YAML files are in this repository:

| File | Schema | Example |
| ---- | ------ | ------- |
| `basics.yaml` | [schemas/basics.json](schemas/basics.json) | [examples/basics.yaml](examples/basics.yaml) |
| `site.yaml` | [schemas/site.json](schemas/site.json) | [examples/site.yaml](examples/site.yaml) |
| `education.yaml` | [schemas/education.json](schemas/education.json) | [examples/education.yaml](examples/education.yaml) |
| `experience.yaml` | [schemas/experience.json](schemas/experience.json) | [examples/experience.yaml](examples/experience.yaml) |
| `interests.yaml` | [schemas/interests.json](schemas/interests.json) | [examples/interests.yaml](examples/interests.yaml) |
| `projects.yaml` | [schemas/projects.json](schemas/projects.json) | [examples/projects.yaml](examples/projects.yaml) |
| `cv-variants.yaml` | [schemas/cv-variants.json](schemas/cv-variants.json) | [examples/cv-variants.yaml](examples/cv-variants.yaml) |
| Blog frontmatter | [schemas/blog-frontmatter.json](schemas/blog-frontmatter.json) | [examples/blog-frontmatter.yaml](examples/blog-frontmatter.yaml) |

If your editor supports the [yaml-language-server](https://github.com/redhat-developer/yaml-language-server), add a schema comment to the top of each file for inline validation and autocompletion:

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/rezelute/launchfolio-docs/main/schemas/basics.json
```

---

## Running with Docker

The recommended way to run Launchfolio is with Docker Compose. Create a `docker-compose.yml`:

```
services:
  launchfolio:
    image: ghcr.io/rezelute/launchfolio:latest
    restart: unless-stopped
    ports:
      - "3000:3000"
    env_file:
      - .env
    volumes:
      - ./content:${PERSONA_DATA_PATH}:ro
```

Then create a .env file in the same directory. Copy .env.example from the repository as a starting point - it documents every available option. At minimum you need:

PERSONA_DATA_PATH=/app/content

PERSONA_DATA_PATH is the path inside the container where your data will be mounted. The compose file uses this same variable for the volume target, so it's the single place you configure the mount. Leave it as `/app/content` unless you have a reason to change it.

Place your persona-data contents in a content/ folder next to docker-compose.yml, then run:

`docker compose up -d`

Your site will be available at http://localhost:3000.

## How Data Loading Works

Understanding how the app loads data will save you confusion when edits don't appear immediately.

YAML files are loaded once at server startup. The app reads every YAML file, validates it, and caches the result in memory. API responses are served from memory with no disk I/O per request - keeping everything fast. The tradeoff: if you edit a YAML file, you must restart the container for the change to take effect.

`docker compose restart launchfolio`.

**Blog posts work differently**. The blog API reads .md files from disk on every request. New posts and edits appear immediately with a browser refresh - **no restart needed**.

Images are served through the IPX image optimizer, which resizes and caches images on first request. New image files appear immediately without a restart.

## Writing Blog Posts

Blog posts are Markdown files in your `blogs/` directory with YAML frontmatter:

```
---
title: "My Post Title"
date: "2025-03-15"
author: Your Name
excerpt: A short description shown on the blog listing page.
tags: [tag-one, tag-two]
cover: /img/blogs/my-post-cover.webp
syndication:
  - https://dev.to/your-post
series:
  slug: my-series
  title: My Series Title
  part: 1
---
```

```
Post content here...
```

A few things to note:

**Always quote the date field** ("2025-03-15"). Unquoted ISO dates are valid YAML date literals and will cause a validation error.

`cover`, `syndication`, and `series` are all optional.

`syndication` is a list of URLs where the post has been cross-posted (e.g. dev.to, Hashnode). Omit or set to `null` if not applicable.

Posts with invalid frontmatter are silently skipped with a warning in the server logs rather than crashing the server.

### Image Guidelines

All images are served through the IPX optimizer, which resizes and converts them on first request and caches the result. You only need one source image per context.

| Where                 | Recommended size    | Notes                                                                                                                                                |
| --------------------- | ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Site logo             | SVG preferred       | Falls back to any raster format. SVG scales perfectly at any size.                                                                                   |
| Blog cover            | 920 × 460 px (2:1)  | Used at full size on the post page and ~400×220 in listing cards.                                                                                    |
| Portfolio cover       | 700 × 394 px (16:9) | Used on both the portfolio and home pages - one image works for both.                                                                                |
| Portfolio screenshots | 1000 px wide        | Shown as 120×80 thumbnails; clicking opens a lightbox at up to 1000 px.                                                                              |
| Favicons              | Generated           | Use <a href="https://realfavicongenerator.net" target="_blank" rel="noopener noreferrer">realfavicongenerator.net</a> to produce all required sizes. |

**Tips:**

- Use `.webp` for photos and screenshots - best quality-to-size ratio.
- Use `.svg` for logos and icons - resolution-independent.
- Keep source images under 2 MB. IPX will compress them for delivery, but very large files slow down first-request processing.
- All image paths in YAML files and blog frontmatter are absolute paths starting from `img/` within your `persona-data` directory:

```
cover: /img/projects/my-project/cover.webp
screenshots:
  - /img/projects/my-project/screenshot-1.webp
```

No uploads or CDN required - images are served directly from your mounted volume.

---

## Quick Reference

| Action                  | What to do                                                       |
| ----------------------- | ---------------------------------------------------------------- |
| Edit YAML content       | Edit the file, restart the container                             |
| Add or edit a blog post | Drop the `.md` file in `blogs/`, refresh the browser             |
| Add or replace an image | Copy the file into `img/`, refresh the browser                   |
| Change configuration    | Update environment variables, restart the container              |
| Debug validation errors | Check container logs on startup - exact field errors are printed |
