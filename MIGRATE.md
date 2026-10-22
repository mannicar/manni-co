# Migrating manni.co from Wix to GitHub Pages (keeping Google Workspace)

Goal: the website at `manni.co` / `www.manni.co` moves from Wix to this repo on
GitHub Pages. Email (`@manni.co` via Google Workspace) must keep working
throughout and after the change.

## Current state (verified 2026-10-22)

DNS is still delegated to the old Google Cloud DNS nameservers (a leftover
from the Google Domains era), not yet to Squarespace's own DNS:

```
NS     ns-cloud-c1.googledomains.com
       ns-cloud-c2.googledomains.com
       ns-cloud-c3.googledomains.com
       ns-cloud-c4.googledomains.com
```

Records currently in that zone:

| Type | Host | Value | Purpose |
|---|---|---|---|
| A | `manni.co` | `23.236.62.147` | Wix hosting (apex) |
| CNAME | `www` | `www158.wixdns.net` | Wix hosting (www) |
| MX | `manni.co` | `1 aspmx.l.google.com`, `5 alt1/alt2.aspmx.l.google.com`, `10 alt3/alt4.aspmx.l.google.com` | **Google Workspace mail — do not touch** |
| TXT | `manni.co` | `v=spf1 include:_spf.google.com ~all` | **SPF for Workspace mail — do not touch** |
| TXT | `google._domainkey` | `v=DKIM1; k=rsa; p=...` | **DKIM for Workspace mail — do not touch** |
| TXT | `manni.co` | `MS=9E5AE7073B20E192C867E476D2B88981CCF42F08` | Microsoft/Bing domain verification — unrelated, leave alone |

No DMARC (`_dmarc`) or CAA records currently exist — not required, optional to add later.

**The only records that need to change are the `A` record at the apex and the
`www` `CNAME`.** Everything else (MX, SPF, DKIM, the MS verification TXT)
stays exactly as-is so Workspace mail is unaffected.

## Where to edit DNS

Log in to the Squarespace Domains panel for `manni.co`
(`domains.squarespace.com` — this is the registrar now, confirmed via WHOIS).
Since the nameservers still point at Google Cloud DNS, check whether
Squarespace's panel lets you edit records directly (it inherited the Google
Domains DNS zone for many accounts). If it does not expose DNS editing for a
domain still on Google's nameservers, you have two options:

1. **Switch nameservers to Squarespace's own** (offered in the same panel),
   then manage all records in Squarespace's DNS editor going forward, or
2. Find whatever Google Cloud DNS project/zone this domain's nameservers
   belong to (likely tied to the original Google Domains/Workspace account)
   and edit records there.

Either way, the end state is the same set of records below — just note which
panel you actually used, since that's where you'll manage DNS from now on.

## Steps

1. **Lower TTLs first (optional but recommended).** If the current A/CNAME
   records have a long TTL, set it to something short (e.g. 300s) a day
   before cutover so the change propagates faster. Not required, just speeds
   up the switch.

2. **Push this repo to GitHub** (ask Claude to do this when ready):
   - Repo: `mannicar/manni-co`
   - Contains `CNAME` file with `manni.co` already in it.

3. **Enable GitHub Pages:**
   - Repo → Settings → Pages
   - Source: `main` branch, `/ (root)`
   - Custom domain: enter `manni.co` → Save (this is also what writes the
     `CNAME` file if it weren't already committed)

4. **Update DNS — apex `A` records.** Replace the existing single Wix `A`
   record (`23.236.62.147`) with four `A` records, all at the apex
   (`manni.co` / `@`):
   ```
   185.199.108.153
   185.199.109.153
   185.199.110.153
   185.199.111.153
   ```
   (Confirmed current via GitHub's own docs, not from memory.)

5. **Update DNS — `www` record.** Replace the Wix `CNAME`
   (`www158.wixdns.net`) with:
   ```
   CNAME   www   mannicar.github.io
   ```

6. **Leave MX, SPF, DKIM, and the MS TXT record untouched.** Do not delete or
   re-create them — Workspace mail depends on them exactly as they are.

7. **Wait for DNS propagation**, then in GitHub repo → Settings → Pages,
   confirm it shows "DNS check successful" for `manni.co`.

8. **Enable "Enforce HTTPS"** in the same Pages settings once GitHub finishes
   provisioning the TLS certificate (can take up to ~24h after DNS check
   passes, usually much faster).

9. **Verify:**
   - `https://manni.co` and `https://www.manni.co` load the new site.
   - Send a test email to/from an `@manni.co` address to confirm Workspace
     mail still works.
   - `dig manni.co MX` still shows the Google mail servers.

## Rollback

If anything goes wrong, the original Wix records are listed in the table
above — re-create the `A` record (`23.236.62.147`) and the `www` `CNAME`
(`www158.wixdns.net`) to point back at Wix. MX/SPF/DKIM were never touched,
so mail is unaffected regardless.
