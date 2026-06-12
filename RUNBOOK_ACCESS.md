# RUNBOOK — Cloudflare Access gate (one login for members + corpus)

**Goal:** one reusable Access policy, **"deppmann-lab,"** that gates
`members.deppmannlab.com` now and `corpus.deppmannlab.com` later — so a lab member
logs in once and reaches both. Free for ≤50 seats.

**Prerequisite:** DNS is on Cloudflare and verified (`RUNBOOK_DNS.md` complete).

> **CONFIRMATION GATES** (need Chris's Cloudflare + Google admin logins): all of
> §2–§6. This runbook is exact enough to click through in ~20 minutes.

> **Two things the gate does NOT do by itself** (read before building):
> 1. The members site has **no auth of its own** — security is 100% the Access
>    app being correctly scoped to the exact hostname. A typo'd host, a detached
>    policy, or the uncovered `*.pages.dev` preview URL serves the whole
>    "do-not-redistribute" portal publicly. Cover the preview URL (§5) and run
>    the negative test (§6) as a hard gate.
> 2. **One-time-PIN is a parallel, lower-assurance door.** Cloudflare emails a
>    PIN to *any* address that satisfies an Allow policy — so a broad
>    "emails ending in @domain" rule + OTP lets anyone who controls *any* mailbox
>    at that domain in, bypassing Google SSO/MFA. Scope OTP to an explicit
>    allowlist only (§3).

---

## 0. Decide the login domain FIRST (the one open question)

"Google login restricted to the lab's Google Workspace domain" needs a confirmed
domain.

- **Note:** the domain's *email* (`deppmannlab.com` MX → `secureserver.net`) is
  **GoDaddy**, not Google. So the lab Google Workspace is on some *other* domain
  (or was set up but email never cut over). Per Chris: the lab "HAD a Workspace —
  may need to refire it."
- **Until the Workspace domain is confirmed (a) which domain, (b) the lab
  controls it, (c) a full inventory of existing accounts/aliases at it — do NOT
  put a `emails ending in @domain` rule live.** A domain rule you don't fully
  control is either inert or an open door (a stale/role mailbox could OTP in).
- **Ship today = Path B only** (OTP + explicit allowlist). Add **Path A** (the
  Google domain rule) once §0 is resolved. The gate works either way.

---

## 1. Turn on Zero Trust (free)

Cloudflare dashboard → **Zero Trust**. Create the org, **Free** plan (≤50 seats).
Team name e.g. `deppmann-lab` → team domain `deppmann-lab.cloudflareaccess.com`.

## 2. Add login methods

Zero Trust → **Settings → Authentication → Login methods**.

- **Google** (generic Google OAuth — you do NOT need the Google Workspace
  connector; the email-domain rule in the policy does the restricting). This is
  the high-assurance path: Google handles SSO + MFA.
- **One-time PIN** (enabled by default) — the email-code path. Treat it as
  **collaborator-only**, gated by an explicit allowlist (§3), never by the domain
  rule.

## 3. Create the reusable Access **Group** — "deppmann-lab"

Zero Trust → **Access → Access Groups → Add a group**, name `deppmann-lab`.
Include rules are OR'd, so scope each by login method so OTP can't ride the
domain rule:

- **Rule 1 (lab members, today + always):** Include **Emails** = explicit
  allowlist of specific people. (This is the only thing live until §0 resolves.)
- **Rule 2 (ADD ONLY AFTER §0 RESOLVES):** Include **Emails ending in**
  `@<LAB-WORKSPACE-DOMAIN>` **AND** Require **Login Method = Google** — so domain
  users must authenticate via Google SSO and **cannot** self-serve an OTP.

This keeps OTP restricted to the named allowlist and forces domain accounts
through Google. Membership = editing this one group.

## 4. Create the Access **Application** for members.deppmannlab.com

Zero Trust → **Access → Applications → Add → Self-hosted**.

- **Name:** Members portal · **Application domain:** `members.deppmannlab.com`
- **Session Duration:** 30 days is the Cloudflare **maximum** (default 24h). For
  a group whose devices leave the lab, **7 days** is a safer default — pick
  deliberately. (Session length is forward-looking; see lost-laptop in §8.)
- **Identity providers:** Google + One-time PIN.
- **Policy → Add:** name `Allow deppmann-lab`, Action **Allow**, Include =
  **Access group → deppmann-lab**.
- **Also cover the preview URL:** the Pages project's `*.pages.dev` hostname is
  NOT protected by the `members.deppmannlab.com` app. Either (a) add a second
  Access app for the `*.pages.dev` host with the same policy, or (b) disable the
  pages.dev subdomain / restrict preview deployments. Don't leave it open.

## 5. Deploy the members site to Cloudflare Pages (the thing being gated)

1. Cloudflare → **Workers & Pages → Create → Pages → Connect to Git** →
   `deppmann/members-deppmannlab`.
2. Build (Astro static): command `npm run build`, output `dist`,
   **Git submodules: On** (fetches the `brand/` submodule).
   - ⚠️ **Submodule caveat:** Cloudflare Pages has historically failed to clone
     submodules referenced by an **HTTPS** `.gitmodules` URL (`could not read
     Username for https://github.com`). `brand` is **public**, which is the
     necessary condition — but **verify the first build actually fetched
     `brand/`**. If it fails, switch `.gitmodules` to the SSH form
     (`git@github.com:deppmann/deppmann-brand.git`) and add a deploy key.
3. **Attach the Access app (§4) BEFORE pointing the custom domain at the project**
   — or accept that the first deploy is briefly public and run the §6 "Blocked"
   test the instant the domain goes live.
4. **Custom domains → Set up a domain →** `members.deppmannlab.com`.

(Repeat 1–2 for the **public** `biochempedia.deppmannlab.com` Pages project, but
with **no** Access app — it's public.)

## 6. Verify the gate end-to-end (hard gate — all four)

- **Allowed (Google):** open `https://members.deppmannlab.com` clean → sign in
  with a lab Google account → portal loads.
- **Allowed (OTP):** request a code to an **allowlisted** email → **confirm the
  code actually ARRIVES** (Cloudflare's page always says "a code was emailed"
  even when it didn't; OTP from `noreply@notify.cloudflare.com` is often
  spam-filtered — pre-allowlist that sender in GoDaddy mail) → portal loads.
- **Blocked (the proof):** a personal `@gmail.com` not in the group → denied.
- **Preview URL:** confirm the `*.pages.dev` URL is also gated or disabled (§4).

## 7. Add corpus.deppmannlab.com later (reuse everything)

When the corpus app has an origin: add its DNS record, create a second
**Self-hosted** app for `corpus.deppmannlab.com` with the **same**
`Allow deppmann-lab` policy **and the same identity providers**. Any extra
*Require* on the corpus app will break single-login continuity (re-prompt). With
identical policy + IdPs, an active members session reaches corpus with no second
prompt.

## 8. Member management & kill switches

- **Add a lab member:** once §0 is resolved, anyone with a lab Google account is
  in via Rule 2 (no action). Before then, add their email to the **allowlist**
  (Rule 1). External collaborator: add their email to the allowlist (OTP).
- **Remove a member — IMPORTANT:**
  - *Allowlist/OTP person:* delete their email from the group. Done.
  - *Domain (Google) account:* deleting from the group does **nothing** if Rule 2
    is live — you must **suspend/delete the Google account** in Google Admin.
    Editing the Cloudflare group does not revoke a domain user.
- **Lost / stolen laptop:** shortening the session length is **NOT retroactive**.
  **Revoke the user's active sessions** (Zero Trust → My Team → Users → revoke)
  and/or set the policy to **Block**.
- **Kill switch (everyone out, now):** set `Allow deppmann-lab` to **Block** or
  pause the Pages project.
- **(Optional) defense-in-depth:** have the app check the
  `Cf-Access-Jwt-Assertion` header server-side so a detached policy fails closed
  rather than serving publicly. (The static Pages site can't do this alone; worth
  it for the corpus app.)
