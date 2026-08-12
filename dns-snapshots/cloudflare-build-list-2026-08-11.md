# Cloudflare record build list — deppmannlab.com

Generated from the GoDaddy zone export of 2026-08-11 18:01:31, which is the
source of truth per RUNBOOK_DNS.md golden rule 1. SOA and NS are omitted:
Cloudflare manages those.

**17 records.** Every one must exist in Cloudflare before the flip.

Three of these were NOT found by the dig sweep, because a sweep can only ask
for names it already knows: `mlp527whylzy7dtndg5n`, `pay`, `pda`.

| Type | Name | Value | Proxy |
|---|---|---|---|
| A | `@` | `75.2.60.5` | **DNS-only (grey)** |
| CNAME | `_domainconnect` | `_domainconnect.gd.domaincontrol.com.` | **DNS-only (grey)** |
| CNAME | `e` | `email.secureserver.net.` | **DNS-only (grey)** |
| CNAME | `email` | `email.secureserver.net.` | **DNS-only (grey)** |
| CNAME | `ftp` | `deppmannlab.com` | **DNS-only (grey)** |
| CNAME | `imap` | `imap.secureserver.net.` | **DNS-only (grey)** |
| CNAME | `mail` | `pop.secureserver.net.` | **DNS-only (grey)** |
| CNAME | `mlp527whylzy7dtndg5n` | `verify.squarespace.com.` | **DNS-only (grey)** |
| CNAME | `mobilemail` | `mobilemail-v01.prod.mesa1.secureserver.net.` | **DNS-only (grey)** |
| CNAME | `pay` | `paylinks.commerce.godaddy.com.` | **DNS-only (grey)** |
| CNAME | `pda` | `mobilemail-v01.prod.mesa1.secureserver.net.` | **DNS-only (grey)** |
| CNAME | `pop` | `pop.secureserver.net.` | **DNS-only (grey)** |
| CNAME | `smtp` | `smtp.secureserver.net.` | **DNS-only (grey)** |
| CNAME | `webmail` | `webmail.secureserver.net.` | **DNS-only (grey)** |
| CNAME | `www` | `deppmannlab.com` | **DNS-only (grey)** |
| MX | `@` | `0 smtp.secureserver.net.` | n/a |
| MX | `@` | `10 mailstore1.secureserver.net.` | n/a |

## Absent — confirmed in both the export and dig

No apex TXT/SPF · no `_dmarc` · no DKIM selectors · no AAAA · no CAA · no `autodiscover`.
Defer any SPF/DKIM/DMARC hardening to a separate change after migration.
