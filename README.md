## What is Launchfolio?
![app](https://github.com/user-attachments/assets/f503140d-a29a-43f3-b535-c114c62dba9c)


Launchfolio is a self-hostable personal portfolio, blog, and CV app. The core idea is simple: **you bring the data, Launchfolio renders it**. Your content lives in a `persona-data` directory you control - YAML files, blog posts, and images. The app reads from it without you ever needing to touch application code.

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
   cv.yaml            # optional - your public CV at /cv
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

| File               | Schema                                                         | Example                                                          |
| ------------------ | -------------------------------------------------------------- | ---------------------------------------------------------------- |
| `basics.yaml`      | [schemas/basics.json](schemas/basics.json)                     | [examples/basics.yaml](examples/basics.yaml)                     |
| `site.yaml`        | [schemas/site.json](schemas/site.json)                         | [examples/site.yaml](examples/site.yaml)                         |
| `education.yaml`   | [schemas/education.json](schemas/education.json)               | [examples/education.yaml](examples/education.yaml)               |
| `experience.yaml`  | [schemas/experience.json](schemas/experience.json)             | [examples/experience.yaml](examples/experience.yaml)             |
| `interests.yaml`   | [schemas/interests.json](schemas/interests.json)               | [examples/interests.yaml](examples/interests.yaml)               |
| `projects.yaml`    | [schemas/projects.json](schemas/projects.json)                 | [examples/projects.yaml](examples/projects.yaml)                 |
| `cv.yaml`          | [schemas/cv.json](schemas/cv.json)                             | [examples/cv.yaml](examples/cv.yaml)                             |
| Blog frontmatter   | [schemas/blog-frontmatter.json](schemas/blog-frontmatter.json) | [examples/blog-frontmatter.yaml](examples/blog-frontmatter.yaml) |

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

**Theme colours (`PRIMARY_COLOR`, `SECONDARY_COLOR`) require a full rebuild.** These are baked into the CSS bundle at build time (used to generate the PrimeVue theme preset and Tailwind utility classes). Changing them in `.env` and restarting the container is not enough — you must rebuild the image. A page refresh or container restart will not pick up colour changes.

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

No uploads or CDN required - images are served directly from your mounted volume. Only images, PDFs and fonts are served - your YAML and Markdown files are never downloadable.

---

## CVs

Launchfolio has two separate CV features.

**Public CV (optional)** - add a `cv.yaml` to show a CV at `/cv`, linked in the footer. It picks entries and highlights from `experience.yaml` and `projects.yaml` by ID, so those files can hold far more than the CV shows. Only the picked highlights are ever sent to the browser. Leave the file out and there's no CV page. See [examples/cv.yaml](examples/cv.yaml).

**CV builder** - go to `/cv/builder` (not linked anywhere) to make one-off CVs as PDFs, e.g. one per job application. Paste YAML or open a `.yaml` file, preview it, and click **Download PDF**. Everything is written in the YAML and nothing is stored or uploaded:

```yaml
header:
  title: Frontend Engineer
  tagline: "Vue/Nuxt specialist"
  name: Jane Smith
  details: [Cambridge, GB] # lines under your name
sections:
  - heading: Summary
    text: "A paragraph of text."
  - heading: Experience
    items:
      - title: My Company # the only required field
        subtitle: Senior Developer
        meta: Jan 2020 - Present # right-hand side
        details: Cambridge, UK · Hybrid
        text: "A short description."
        bullets: ["An achievement."]
        tags: [Vue, TypeScript]
  - heading: Interests
    tags: [Open Source, Nature]
```

Every section has a `heading` plus any mix of `text`, `items` and `tags`. All values are text - quote anything YAML would read as a number, e.g. `meta: "2026"`. Keep your CV files somewhere private, not in your persona-data directory.

### Upgrading from cv-variants.yaml

`cv-variants.yaml` has been replaced, and the server will now ignore it:

1. To keep a public CV, copy one variant into a new `cv.yaml` and remove its `id` and `listed` lines. It's now served at `/cv` instead of `/cv/<id>`.
2. Move any custom highlights (`{ id, text }` entries) into `experience.yaml` or `projects.yaml`, or into a builder CV - `cv.yaml` only accepts highlight IDs.
3. For other variants, rebuild them as builder YAML files.
4. Delete `cv-variants.yaml`.

---

## Quick Reference

| Action                  | What to do                                                       |
| ----------------------- | ---------------------------------------------------------------- |
| Edit YAML content       | Edit the file, restart the container                             |
| Add or edit a blog post | Drop the `.md` file in `blogs/`, refresh the browser             |
| Add or replace an image | Copy the file into `img/`, refresh the browser                   |
| Change configuration    | Update environment variables, restart the container              |
| Change theme colours    | Update `PRIMARY_COLOR` / `SECONDARY_COLOR`, **rebuild the image** (restart alone is not enough) |
| Debug validation errors | Check container logs on startup - exact field errors are printed |
