# @deppmann/brand

Shared design tokens, footer, and marks for the **deppmannlab.com ecosystem**.
The single source of truth that keeps four parallel builds looking like one place.

- `tokens.css` — all design tokens (`:root` custom properties) + the webfont import
- `Footer.astro` — the shared, portable footer (carries the student + inspiration credit lines; `internal` flag for gated surfaces)
- `assets/logo.svg`, `assets/wordmark.svg`, `assets/favicon.svg` — the marks
- `BRAND.md` — the brand guide (palette, type, usage)
- `DECISIONS.md` — host / DNS / gate decisions + **kill-switch locations** (read this before touching infra)
- `RUNBOOK_DNS.md` — exact, safe steps to move DNS GoDaddy → Cloudflare without breaking email
- `RUNBOOK_ACCESS.md` — exact steps to stand up the Cloudflare Access gate (Google-domain login + email fallback)

## Hosting & gate decisions (the short version)

- Public `deppmannlab.com` + `www` stay on **Netlify**. DNS moves to Cloudflare but those records stay **DNS-only (grey cloud)** pointing at Netlify. **Never proxy Netlify through Cloudflare** (breaks Let's Encrypt renewal).
- Only the **gated subdomains** (`members.`, `corpus.`) run on **Cloudflare Pages behind Cloudflare Access**. `biochempedia.` is public Cloudflare Pages.
- One reusable Access policy, **"deppmann-lab"**, gates members + corpus with one login (Google Workspace domain, with a small email-allowlist exception).

## How to consume this package (git submodule)

We share by **git submodule**, not a registry: no publish step, pinned by commit
SHA, and both Netlify and Cloudflare Pages initialize submodules automatically on
build.

**1. Add the submodule** (once, in a consumer repo):

```bash
git submodule add https://github.com/deppmann/deppmann-brand.git brand
git commit -m "Add deppmann-brand submodule"
```

**2. Clone / pull with submodules:**

```bash
git clone --recurse-submodules <repo>
# or, in an existing checkout:
git submodule update --init --recursive
```

**3. Use it.** In an Astro layout:

```astro
---
import Footer from "../../brand/Footer.astro";   // path is relative to the file
---
<head>
  <link rel="icon" type="image/svg+xml" href="/favicon.svg" />
</head>
<body>
  <slot />
  <Footer />
</body>

<style is:global>
  @import "../../brand/tokens.css";   /* load tokens BEFORE your own CSS */
  @import "../styles/global.css";
</style>
```

Copy `assets/favicon.svg` into the consumer's `public/favicon.svg` (favicons must
live at the site root; they can't be served from the submodule path).

**Updating the brand:** edit files here, commit, push. In each consumer:
`git submodule update --remote brand && git add brand && git commit`. The
build platforms re-fetch the new pinned SHA on next deploy.

### Publish order (do this FIRST)

This repo is the root of the dependency graph — every consumer pins a brand SHA.
**Push `deppmann-brand` (public) to GitHub and confirm the pinned commit is
fetchable BEFORE pushing/merging or connecting any consumer.** Otherwise the
first consumer build fails on `git submodule update --init` (the commit won't
exist on the remote). Sequence: **brand → consumers → connect Netlify/CF Pages.**

### Build-platform notes

- **Netlify** initializes submodules automatically when the submodule repo is
  reachable (a *public* submodule needs no extra key — keep this repo public).
- **Cloudflare Pages** runs `git submodule update --init` during build. It has
  historically failed to clone submodules via an **HTTPS** `.gitmodules` URL even
  for public repos — **verify the first build fetched `brand/`**; if not, switch
  `.gitmodules` to the SSH form (`git@github.com:deppmann/deppmann-brand.git`)
  plus a deploy key.

> Mechanism rationale and the alternative (npm-from-git) are recorded in
> `DECISIONS.md`.
