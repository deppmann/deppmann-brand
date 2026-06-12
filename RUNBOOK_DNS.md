# RUNBOOK — Move DNS GoDaddy → Cloudflare (without breaking email or the live site)

**Goal:** put `deppmannlab.com` on Cloudflare DNS so we can use free Cloudflare
Access on the gated subdomains — while the public site stays on Netlify and email
(GoDaddy) keeps flowing for both servers AND mail clients.

**Golden rules**
1. **The GoDaddy zone EXPORT is the only source of truth.** A `dig` sweep and
   Cloudflare's auto-scan both UNDER-CAPTURE (this was proven during planning —
   an early hand-built table missed seven live records). Replicate the export
   record-for-record before flipping nameservers.
2. **Apex + www stay DNS-only (grey cloud) pointing at Netlify.** Never turn on
   Cloudflare's orange-cloud proxy for them — double-CDN breaks Netlify's
   Let's Encrypt renewal. (Apex and www share ONE Netlify cert, so proxying
   *either* one risks the whole cert's renewal.)
3. **"Don't break email" = MX **and** the mail-client CNAMEs.** Preserving MX
   keeps server-to-server delivery, but Outlook/Apple Mail/phones/webmail connect
   to `imap.`, `smtp.`, `pop.`, `mail.`, `webmail.` — drop those and every client
   stops connecting even though deliverability "tests green."
4. **Reversible, but NOT fast.** Until you flip nameservers, nothing changes.
   The flip — and a rollback — are governed by the parent `.com` delegation TTL
   of **48 hours**, not the zone's 1h SOA. So the real guarantee is the pre-flip
   parity check (§4), not the rollback.

> **CONFIRMATION GATES** (need Chris): (A) adding the Cloudflare zone — needs the
> Cloudflare account login; (B) the GoDaddy nameserver change. Everything before
> A is verification you can do now.

---

## 0. Current zone — captured 2026-06-12 against GoDaddy authoritative NS (ns35.domaincontrol.com)

These resolve today and **must all exist in Cloudflare before the flip.**
Treat this table as a CHECKLIST, not the source of truth — reconcile it against
the GoDaddy **zone export** (§1), which is authoritative.

| Type | Name | Value | Proxy | Notes |
|---|---|---|---|---|
| A | `@` | `75.2.60.5` | **DNS-only** | Netlify apex load balancer |
| CNAME | `www` | `deppmannlab.com` | **DNS-only** | redirects to apex via Netlify |
| MX | `@` | `smtp.secureserver.net` (pri **0**) | n/a | **GoDaddy email — DO NOT CHANGE** |
| MX | `@` | `mailstore1.secureserver.net` (pri **10**) | n/a | **GoDaddy email — DO NOT CHANGE** |
| CNAME | `email` | `email.secureserver.net` | DNS-only | GoDaddy webmail/login |
| CNAME | `e` | `email.secureserver.net` | DNS-only | GoDaddy email alias |
| CNAME | `mail` | `pop.secureserver.net` | DNS-only | **mail client** |
| CNAME | `smtp` | `smtp.secureserver.net` | DNS-only | **mail client (outgoing)** |
| CNAME | `pop` | `pop.secureserver.net` | DNS-only | **mail client (POP)** |
| CNAME | `imap` | `imap.secureserver.net` | DNS-only | **mail client (IMAP)** |
| CNAME | `webmail` | `webmail.secureserver.net` | DNS-only | **webmail** |
| CNAME | `mobilemail` | `mobilemail-v01.prod.mesa1.secureserver.net` | DNS-only | **mobile mail** |
| CNAME | `ftp` | `deppmannlab.com` | DNS-only | GoDaddy default |
| CNAME | `_domainconnect` | `_domainconnect.gd.domaincontrol.com` | DNS-only | GoDaddy Domain Connect |

**Replicate ALL of the above. Do not make per-record "is this needed?" judgment
calls — selective dropping is exactly how mail clients break.**

**Not found via public dig (CONFIRM against the export — do NOT assume absent):**
no apex `TXT`/SPF, no `_dmarc`, no DKIM selectors, no `AAAA`, no `CAA`,
no `autodiscover`. (The early hand-table already proved a dig-based "absent"
claim unreliable for CNAMEs, so verify these from the export, not from dig.)

> ⚠️ **Email is GoDaddy, not Google.** The domain's mail (MX + the
> `secureserver.net` client CNAMEs) is GoDaddy. It is independent from the
> *lab's Google Workspace* used for the Access login. The DNS move touches
> neither — provided every `secureserver.net` record above is replicated exactly.
> Defer any SPF/DKIM/DMARC hardening to a **separate** change after migration.

---

## 1. Pre-flight (BEFORE adding the Cloudflare zone) — no account needed

**1a. Export the GoDaddy zone (MANDATORY — this is the source of truth).**
GoDaddy → *My Products → Domains → deppmannlab.com → DNS → Manage Zone → ⋯ →
Export Zone File*. Save it as `godaddy-zone-export.txt`. Everything below is
checked against THIS file.

**1b. Snapshot live resolution** (sanity cross-check of the export):

```bash
D=deppmannlab.com; NS=ns35.domaincontrol.com
for t in NS SOA A AAAA MX TXT CAA; do echo "== $t =="; dig +short $t $D @$NS; done
for h in www email e mail smtp pop imap webmail mobilemail ftp _domainconnect autodiscover; do
  printf "%-14s -> " "$h"; dig +short CNAME $h.$D @$NS
done
echo "== _dmarc =="; dig +short TXT _dmarc.$D @$NS
for s in selector1 selector2 default google k1 k2; do
  printf "DKIM %-9s -> " "$s"; dig +short CNAME ${s}._domainkey.$D @$NS; dig +short TXT ${s}._domainkey.$D @$NS
done
```

Diff this against `godaddy-zone-export.txt`. The export wins. Any record in
either that isn't in the §0 checklist must still be carried into Cloudflare.

## 2. (Chris) Add the zone to Cloudflare — GATE A

1. Cloudflare dashboard → **Add a site** → `deppmannlab.com` → **Free** plan.
2. Let Cloudflare auto-scan. **Do not flip nameservers yet.** Note the two
   assigned Cloudflare nameservers (e.g. `xxx.ns.cloudflare.com`) for §5.

## 3. Reconcile records in Cloudflare (before the flip)

- Compare Cloudflare's imported records against **`godaddy-zone-export.txt`**
  (not the auto-scan, not the §0 table). Add every missing record; fix any wrong
  value. Pay special attention to the **mail-client CNAMEs** (`mail`, `smtp`,
  `pop`, `imap`, `webmail`, `e`, `mobilemail`) — auto-scan commonly misses them.
- Set **A `@`** and **CNAME `www`** to **DNS-only (grey cloud)**. Confirm A = `75.2.60.5`.
- Confirm both **MX** records exactly (`smtp` pri 0, `mailstore1` pri 10).
- Replicate the remaining CNAMEs (grey cloud), including `_domainconnect` and `ftp`.
- Leave TTLs on **Auto**.
- ✅ Pre-flip checklist: every export record present · apex A right + DNS-only ·
  www CNAME right + DNS-only · both MX present · ALL seven mail CNAMEs present ·
  no orange clouds anywhere.

## 4. HARD GATE — full record-for-record parity vs Cloudflare NS (before the flip)

This is the real safety guarantee (the rollback is slow — §7). Query **every
name from the export** against both the GoDaddy NS and the Cloudflare-assigned
NS and require identical answers:

```bash
D=deppmannlab.com; OLD=ns35.domaincontrol.com; CF=xxx.ns.cloudflare.com  # your CF NS from §2
fail=0
# apex + MX
for q in "A @" "MX @"; do set -- $q; t=$1
  a=$(dig +short $t $D @$OLD | sort); b=$(dig +short $t $D @$CF | sort)
  [ "$a" = "$b" ] && echo "OK   $t @" || { echo "DIFF $t @: GoDaddy[$a] CF[$b]"; fail=1; }
done
# every host CNAME (add any extra labels found in the export)
for h in www email e mail smtp pop imap webmail mobilemail ftp _domainconnect; do
  a=$(dig +short CNAME $h.$D @$OLD); b=$(dig +short CNAME $h.$D @$CF)
  [ "$a" = "$b" ] && echo "OK   CNAME $h" || { echo "DIFF CNAME $h: GoDaddy[$a] CF[$b]"; fail=1; }
done
[ $fail -eq 0 ] && echo "PARITY OK — safe to flip" || echo "DO NOT FLIP — fix diffs first"
```

**Do not proceed to §5 until this prints `PARITY OK`.**

## 5. (Chris) Flip nameservers at GoDaddy — GATE B

GoDaddy → domain → **Nameservers → Change → Enter my own nameservers** → replace
`ns35.domaincontrol.com` / `ns36.domaincontrol.com` with the two Cloudflare
nameservers from §2. Save.

> **Timing reality:** the `.com` parent delegation TTL is **48 hours**. Most
> resolvers pick up the change within an hour, but some cache the old delegation
> for up to ~48h. Plan the flip for a low-stakes window and expect a 48h tail.

> **Do NOT delete or edit the GoDaddy-hosted zone after the flip.** It is your
> rollback source of truth (§7). Leave it untouched for at least a week after a
> verified-green flip.

## 6. Post-flip verification (do all of these)

```bash
D=deppmannlab.com
dig +short NS $D                       # expect the two *.ns.cloudflare.com
dig +short A  $D                       # expect 75.2.60.5
dig +short MX $D                       # expect 0 smtp.secureserver.net / 10 mailstore1...
for h in imap smtp pop mail webmail email; do printf "%-8s " $h; dig +short CNAME $h.$D; done  # all -> *.secureserver.net
curl -sI https://$D | head -n 1        # expect HTTP/2 200 (Netlify serving)
curl -svo /dev/null https://$D 2>&1 | grep -Ei 'subject:|expire'   # cert valid, covers apex+www
```

- Open `https://deppmannlab.com` and `https://www.deppmannlab.com` — both load,
  padlock valid, no cert warning.
- **Email — test clients, not just delivery:** confirm `imap/smtp/pop/mail/webmail`
  resolve to their `secureserver.net` targets (above), THEN **reconnect a real
  mail client** (quit + reopen Apple Mail/Outlook, or send+receive from a phone)
  and load `webmail.deppmannlab.com`. A single webmail send is not enough — it
  can pass on a cached session while client reconnection is broken.
- In Netlify: domain still active, certificate valid.

## 7. Rollback (correct, but slow — up to 48h)

GoDaddy → Nameservers → set back to `ns35.domaincontrol.com` /
`ns36.domaincontrol.com`. Because you never edited the GoDaddy zone, the old
records return — but resolvers that cached the Cloudflare delegation may take up
to ~48h to revert. This is why §4 (pre-flip parity) is the real safeguard: if §4
passed, you should never need this. If you do roll back, expect a degraded tail.

## 8. Only after the public site + email (servers AND clients) verify green

Proceed to `RUNBOOK_ACCESS.md` for the gated subdomains (`members.`, later
`corpus.`) on Cloudflare Pages behind Access, and the public `biochempedia.`
Pages project. Do not revisit apex/www.
