# Ecosystem Decisions & Kill-Switch Map

The operational source of truth for the deppmannlab.com ecosystem. If you touch
hosting, DNS, or the gate, read this first. Last updated 2026-08-12 (teaching site
launched on its own domain — see the note under Architecture). Before that,
2026-08-08 (portal build: D11–D13, D3 amended).

## Architecture

```
                    deppmannlab.com  (PUBLIC — stays on Netlify, DNS-only on Cloudflare)
                     ├── /podcast      → "We Are Biochemistry" (Spotify embed + RSS)
                     └── /book         → The Molecule Hunters (KDP + CC-BY-NC mirror)

  ── Cloudflare Access · group "deppmann-lab" · 30-day session ───── GATED ──
                     ├── members.deppmannlab.com   → lab portal (Workers + D1 + R2)
                     │      colony · antibodies · panels · images · protocols
                     └── corpus.deppmannlab.com    → Federated Corpus app (LATER)

                    biochemistrypedia.com  (PUBLIC — its own zone, Cloudflare Worker)
                     └── www.biochemistrypedia.com  → same Worker; canonical is the apex

  Shared: deppmann-brand (this repo) · one Access group · one RAG+citation pattern
```

> **Teaching site moved out of the subdomain plan (2026-08-12).** Biochemistrypedia
> was going to be `biochempedia.deppmannlab.com`. It launched instead on its own
> Cloudflare-registered domain, `biochemistrypedia.com`, because that zone already
> sat in the same Cloudflare account as the Worker — so binding it needed no DNS
> change at GoDaddy, where `deppmannlab.com` is still authoritative. It is a
> **Worker** (Workers Builds, git-connected to `main`), not a Pages project; the
> origin is `biochempedia.deppmann.workers.dev`. The repo keeps the older name
> `deppmann/biochempedia`.

## Decisions (do not relitigate)

| # | Decision | Why |
|---|---|---|
| D1 | **DNS → Cloudflare**, public records **DNS-only** at Netlify | Free Access needs Cloudflare DNS; keep Netlify serving the public site |
| D2 | **Never proxy Netlify through Cloudflare** (no orange cloud on apex/www) | Double-CDN breaks Netlify's Let's Encrypt renewal (Netlify support guidance) |
| D3 | Public site host = **Netlify**; gated subdomains = **Cloudflare**. **Amended 2026-08-08: new surfaces target Workers with Static Assets, not Pages.** **Amended 2026-08-12: the teaching site is no longer a subdomain — it launched as a Worker on its own domain, `biochemistrypedia.com`** | Don't migrate a working site; put new surfaces where the gate is. Cloudflare's current guidance is that new projects start on Workers and that all platform investment now goes there — and Workers is what lets one deployment carry the site plus its D1 and R2 bindings. Same account, same Access gate, same custom domain; only the deploy target changes. Existing Pages projects keep working |
| D4 | Auth = **Cloudflare Access**, one reusable group **"deppmann-lab"** | One login for members + corpus; ≤50 seats = free |
| D5 | Login = **Google (lab Workspace domain, Login-Method=Google required)** + **One-time-PIN scoped to an explicit allowlist only** | Auto-onboard lab accounts via SSO/MFA; OTP covers outside collaborators and the not-yet-refired-Workspace case — but OTP must NOT ride the domain rule (see `RUNBOOK_ACCESS.md` §3) or anyone with a domain mailbox could bypass SSO |
| D6 | **30-day** Access session | Low-friction for a small trusted lab |
| D7 | Subdomains: `members.` + `corpus.` gated; `/podcast` + `/book` are pages on the public site. Superseded in part 2026-08-12: the teaching site left the subdomain plan for its own public zone, `biochemistrypedia.com` | Front-door model |
| D8 | **4 repos + 1 brand package**, all under the `deppmann` GitHub account | Clean parallel ownership / student handoff |
| D9 | Brand shared via **git submodule** (`brand/`), pinned by SHA | No registry/publish step; Netlify + CF Pages both auto-init submodules; immutable pin |
| D10 | API-backed surfaces (corpus, tutor): **server-side key, hard $ cap, kill switch, named student owner** | Sabbatical maintenance is the real risk, not the build |
| D11 | The members app **verifies `Cf-Access-Jwt-Assertion` server-side on every request** and fails closed | `RUNBOOK_ACCESS.md` §8 listed this as optional defence-in-depth. It stopped being optional when the portal became the colony census: a detached or mis-scoped policy must return 403, not serve the lab's animal records to anyone who finds the URL |
| D12 | Public protocols travel as a **JSON snapshot in a public R2 bucket + a Netlify build hook**. The public site keeps **no database path** — no Access Bypass, no read endpoint | A snapshot cannot leak a row it does not contain. The alternative puts a public door on a database holding animal records with only a `WHERE` clause in between. The cost is staleness until the next build, which for protocol text is not a cost |
| D13 | Data layer for gated apps = **Cloudflare D1 + R2**, inside the existing account | No second vendor and no second login beside Access, and the Access JWT already supplies the verified identity every audit row needs. Needs Workers Paid ($5/mo): D1 free caps a database at 500 MB |

**D9 alternative considered:** npm-install-from-git
(`"@deppmann/brand": "github:deppmann/deppmann-brand#<sha>"`). Also registry-free
and works on both platforms; rejected for Phase 0 only because the submodule gives
plain relative imports (`brand/tokens.css`) with no package-resolution step in
CSS `@import`. Either is fine; if submodule DX becomes painful, switch consumers
to the npm-from-git form — same source repo.

## Repos

| Repo | Visibility | Host | Gate | Consumes brand? |
|---|---|---|---|---|
| `deppmann/deppmannlab-website` | **private** (today) | Netlify | public site | yes |
| `deppmann/deppmann-brand` (this) | **public** | n/a (package) | n/a | — |
| `deppmann/members-deppmannlab` | private | **Cloudflare Workers** (D1 + R2) | Access | yes |
| `deppmann/biochempedia` | public | **Cloudflare Worker** | public | yes |
| `deppmann/federated-corpus` | private | TBD (later) | Access | no (Python app) |
| `deppmann/molecule-hunters` | public | n/a (CC-BY-NC mirror) | public | no (content) |

> `deppmann-brand` **must be public** so Netlify/CF Pages can init the `brand/`
> submodule without a deploy key. The *consumer* repos can be private (the
> public site repo currently is) — what matters for keyless submodule fetch is
> that the **submodule** repo is public.

## Confirmation gates (need Chris)

These are NOT done automatically. Each requires Chris's explicit go-ahead and, in
most cases, his account login:

0. **STEP 0 — push `deppmann-brand` to GitHub as a PUBLIC repo FIRST**, and
   confirm the pinned commit is fetchable (`git ls-remote https://github.com/deppmann/deppmann-brand.git`
   lists it). Brand is the root of the dependency graph: every consumer pins a
   brand SHA via submodule, so any consumer pushed/built before brand exists will
   fail submodule init. **Order: brand → consumers → connect Netlify/CF Pages.**
1. **Create the other GitHub repos** + push Phase-0 branches + open PRs (under his account).
2. **DNS:** add Cloudflare zone (Gate A) and flip GoDaddy nameservers (Gate B) — `RUNBOOK_DNS.md`.
3. **Access:** Zero Trust org, login methods, the `deppmann-lab` group, the members application — `RUNBOOK_ACCESS.md`.
4. **Cloudflare:** deploy `members-deppmannlab` as a **Worker** (needs a D1 database, two R2 buckets and a KV namespace first — see that repo's `SETUP.md`); `biochempedia` is already deployed as a Worker. Set custom domains.
4b. **Workers Paid ($5/mo)** for the portal — D1 free caps a database at 500 MB (D13).
5. **Merge** any PR to a default branch (Claude never pushes to default branches).

## Kill-switch map (how to turn each thing off, fast)

| Surface | Kill switch | Where |
|---|---|---|
| Public site `deppmannlab.com` | Stop the Netlify site / lock deploys | Netlify dashboard → site → Deploys / General |
| `/podcast`, `/book` | Revert the page, redeploy | website repo |
| `members.` (and `corpus.`) gate | Set the `Allow deppmann-lab` policy to **Block**, or disable the Worker | Cloudflare Zero Trust → Access → Applications; Workers & Pages |
| Who's in the lab | Edit Access group **`deppmann-lab`** Include list | Cloudflare Zero Trust → Access → Access Groups |
| Portal writes (keep reads) | Set every `users.role` to `viewer` | D1: `UPDATE users SET role='viewer'` |
| One portal account | `UPDATE users SET active=0 WHERE email=…` | D1 (independent of the Access group) |
| Public protocols | Set each protocol back to `private`, then Republish | Portal → Protocols |
| `biochemistrypedia.com` (Biochemistrypedia) | Remove the custom domains from the Worker, or disable the Worker | Cloudflare → Workers & Pages → `biochempedia` → Domains |
| Corpus / tutor API spend | Revoke the server-side API key; it has a hard monthly cap | provider dashboard; key stored server-side only, never in git |
| Analytics | Plausible script tag (`pa-GJ6PGXY_lPdHXDDJMukci`) in `BaseLayout.astro` | website repo |
| DNS (whole domain) | Point GoDaddy nameservers back to `ns35/36.domaincontrol.com` | GoDaddy → Nameservers (rollback, `RUNBOOK_DNS.md` §7) |

## Cost posture

Domain owned · Cloudflare DNS+Access **$0** (<50 users) · Netlify + CF Pages free
tiers **$0** · Podcast TTS **<$50/season** · Corpus/tutor API governed by the
grant's $3,500 line (RAG + Haiku + prompt caching, not fine-tuning), separate
small lab key capped at **$50/mo**. The real cost is human maintenance time —
hence D10 (every AI surface: cap + kill switch + named owner).

## Runbooks in this repo

- `RUNBOOK_DNS.md` — GoDaddy → Cloudflare migration (captured zone, exact steps, rollback)
- `RUNBOOK_ACCESS.md` — Cloudflare Access gate (Google + OTP fallback, the `deppmann-lab` group)
- `BRAND.md` — palette / type / marks / footer / credit lines
