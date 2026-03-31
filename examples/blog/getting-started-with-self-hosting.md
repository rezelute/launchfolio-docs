---
title: "Getting Started with Self-Hosting"
date: "2025-03-15"
author: Jane Doe
excerpt: "A practical guide to running your own services at home, starting with the essentials and why it matters for your privacy."
tags: [self-hosting, privacy, docker, linux]
cover: /img/blogs/self-hosting.webp
syndication: null
---

## Why Self-Host?

Every app you use on someone else's server is data you don't control. Self-hosting flips that equation — you own the server, you own the data, and you decide who has access.

It sounds daunting, but modern tooling has made it surprisingly approachable. If you can run `docker compose up`, you can self-host.

## Where to Start

The best first step is picking one service to replace. A good candidate is something you already pay for or something that handles sensitive data.

Popular starting points:

- **Nextcloud** — file sync and sharing (replaces Google Drive / Dropbox)
- **Immich** — photo library (replaces Google Photos)
- **Vaultwarden** — password manager (replaces LastPass / 1Password)

## The Basics You'll Need

Before running anything, get comfortable with three things:

1. **Docker and Docker Compose** — almost every self-hosted project ships a `compose.yml`
2. **A reverse proxy** — Caddy or Nginx Proxy Manager to route traffic and handle HTTPS
3. **A domain name** — a cheap `.dev` or `.net` domain is all you need

Once those are in place, deploying a new service is usually a matter of copying a compose file and running it.

## Security First

Exposing services to the internet comes with responsibility. A few non-negotiables:

- Keep everything updated — subscribe to security advisories for anything you run
- Use strong, unique passwords and enable 2FA wherever possible
- Restrict access with a VPN (Tailscale makes this very easy) rather than exposing everything publicly

Self-hosting is a journey, not a destination. Start small, learn as you go, and enjoy owning your stack.
