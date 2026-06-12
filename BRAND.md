# Deppmann Lab — Brand

The shared visual language for the deppmannlab.com ecosystem. One palette,
one type program, one footer, one mark — so four surfaces built in parallel
read as one place.

**Surfaces that consume this package**

| Surface | Host | Gate | Repo |
|---|---|---|---|
| `deppmannlab.com` (public lab site) | Netlify | public | `deppmann/deppmannlab-website` |
| `members.deppmannlab.com` (lab portal) | Cloudflare Pages | Cloudflare Access | `deppmann/members-deppmannlab` |
| `biochempedia.deppmannlab.com` (teaching) | Cloudflare Pages | public | `deppmann/biochempedia` |
| `/podcast`, `/book` | pages on the public site | public | (in the website repo) |
| `corpus.deppmannlab.com` (corpus app) | TBD (later) | Cloudflare Access | `deppmann/federated-corpus` |

---

## Palette

The lab's identity is **deep navy + warm gold + teal** on a soft cream page —
a neuron lit from inside.

| Token | Hex | Use |
|---|---|---|
| `--color-navy` | `#1B2A4A` | primary text on light, dark sections, headings |
| `--color-navy-light` | `#2C3E6B` | gradient partner for navy |
| `--color-navy-dark` | `#111D35` | footer, deepest backgrounds |
| `--color-gold` | `#D4A843` | accents, section labels, primary buttons, links of emphasis |
| `--color-gold-light` | `#E8C76B` | gradient partner / hover |
| `--color-gold-dark` | `#B8912E` | gold text on light (contrast-safe) |
| `--color-teal` | `#2A7B88` | links, interactive accents, icons |
| `--color-teal-light` | `#3A9DAD` | focus rings, gradient partner |
| `--color-teal-dark` | `#1E5C66` | link hover, teal text on light |
| `--color-bg` | `#F8F6F2` | page background (cream) |
| `--color-bg-alt` | `#F0EDE7` | alternating bands |
| `--color-text` | `#2D2D2D` | body copy |
| `--color-text-light` | `#5A5A5A` | secondary copy |
| `--color-text-muted` | `#8A8A8A` | captions, sublines |
| `--color-border` / `--color-border-light` | `#E2DFD8` / `#EDEAE4` | hairlines, card edges |

Gradients are part of the brand, not decoration: navy→teal for the mark and
dark sections, gold→gold-light for primary buttons, teal→gold for accent rules.

## Type

- **Headings:** Playfair Display (serif, 600/700). Literary, a little warm.
- **Body / UI:** Inter (300–700).
- **Mono:** SF Mono / Fira.

Both webfonts load via the `@import` at the top of `tokens.css` — consumers get
them automatically. Use the fluid scale (`--text-xs` … `--text-5xl`); never
hardcode px font sizes.

## Spacing, layout, motion

Use the `--space-*` scale, `--max-width*`, `--border-radius*`, `--shadow-*`, and
`--transition-*` tokens rather than literals. The spring transition
(`--transition-spring`) is the house easing for hover lifts. Everything must
respect `prefers-reduced-motion` (the public site already does).

## Marks

- `assets/logo.svg` — the neuron mark, square, transparent. Use at ≥32px.
- `assets/favicon.svg` — same mark, optimized as a favicon. Copy to each site's
  `public/favicon.svg`.
- `assets/wordmark.svg` — horizontal lockup (mark + "Deppmann Lab" +
  "University of Virginia"). Use in headers/letterhead where space allows.

Clear space around the mark = the radius of the soma (the central circle). Don't
recolor the mark; it carries the navy→teal→gold story on its own. On dark
backgrounds the mark already reads (the soma gradient + gold core glow).

## Footer (`Footer.astro`)

The one footer for every surface. It carries two required lines:

- **Student credit** — names the trainees who build and keep these surfaces
  alive. This is deliberate: in sabbatical, every AI surface has a named student
  owner, and the people who do the work get their names on it.
- **Inspiration credit** — the one-line "why" behind the ecosystem.

Pass `internal={true}` on gated surfaces (members, corpus) to render the
**"Lab-internal — do not redistribute"** banner.

### Credit lines — CONFIRM OR REPLACE

The package ships sensible defaults in Chris's voice, but these are placeholders
until Chris signs off (they appear on every page, in his name):

- Student credit (default):
  > "Designed and built by the trainees of the Deppmann Lab."
  Replace with named trainees per surface if desired, e.g.
  *"Members portal built by [Name]; corpus app by [Name]."*
- Inspiration credit (default):
  > "Built on a simple idea: science is a team sport, and history should
  > remember the generous, not only the brilliant."
  This deliberately echoes the *moral-memory* thesis of *The Molecule Hunters*.
  If the inspiration is meant to credit a specific person or source, set it here.

Override per surface via props:

```astro
<Footer
  studentCredit="Members portal by Jane Doe • Corpus by John Roe"
  inspirationCredit="For the students who do the work history forgets."
/>
```

## Voice on the surfaces

Short, direct, a little literary — the same register as the lab site and the
SABBATICAL_OS docs. Prose written *as Chris* (onboarding copy, announcements,
letters) must go through the `deppmann-voice` skill first. This package owns how
the surfaces *look*; `deppmann-voice` owns how they *read*.

## How to consume

See [`README.md`](./README.md). Short version: add this repo as a `brand/` git
submodule, import `brand/tokens.css` before your own CSS, and use
`brand/Footer.astro`. Edit tokens **here**, never in a consumer.
