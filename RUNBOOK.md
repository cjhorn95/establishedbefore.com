# establishedbefore.com — rebuild-from-zero runbook

Public file. Contains nothing that is not already visible on the internet. No credentials,
no business documents, no personal emails. If you need a secret, it lives in the operator's
credential vault, never here.

Purpose: any agent or person with `gh` and a GoDaddy DNS login can recreate this site,
end to end, from this file alone. Machine-readable twin: `site.manifest.yaml`
(schema `urn:fleet:site-manifest:v1`; over the fleet bus it travels as a `data` part,
media type `application/vnd.fleet.site-manifest+yaml`).

## 1. What this is

One static landing page for The Established Before Co., served by GitHub Pages from this
repo, at `https://establishedbefore.com`. `.co` and `.shop` forward to it. The page's CTA
sends visitors to the Etsy shop.

| Item | Value |
|---|---|
| Live URL | https://establishedbefore.com/ |
| Repo | https://github.com/cjhorn95/establishedbefore.com (public) |
| Pages fallback URL | https://cjhorn95.github.io/establishedbefore.com/ (301s to the domain) |
| Source of record | this repo, `main`, `index.html` — the public site repo is the canonical copy of the published page |
| Page sha256 | `b372c9f505421f0ad340e41fca373e159ec3118574ebdb99a7c9e2b0eb4fb34e` (index.html, 87,092 bytes) — fleet-wide hash algorithm; md5 `6daf4ad0ecd95a50a9a9eefe0984f574` is the approval-record cross-check only |
| OG image | `og.png` 1200×630, 514,405 bytes, referenced as `https://establishedbefore.com/og.png` |
| CTA target | https://www.etsy.com/shop/EstablishedBefore |
| Contact shown on page | none beyond the shop link |

## 2. Repo layout (root of `main`, nothing else — ever)

```
index.html      the page (copied verbatim from the source of record)
og.png          Open Graph image
CNAME           one line: establishedbefore.com
.nojekyll       empty; disables Jekyll so files serve as-is
README.md       one line: site only, rebuild instructions in RUNBOOK.md
RUNBOOK.md      this file
site.manifest.yaml
```

Rule: this repo holds the published page and its rebuild docs only. Design work, copy
decisions, and business material stay in the fleet's private repos. Do not add folders.

## 3. Rebuild steps

### 3.1 Repo
```bash
mkdir establishedbefore.com && cd establishedbefore.com && git init -b main
# get the page: clone this repo (or the fleet's mirror of it) and take index.html + og.png as-is
printf 'establishedbefore.com\n' > CNAME
: > .nojekyll
printf '# establishedbefore.com\n\nSite only; rebuild instructions in RUNBOOK.md.\n' > README.md
git add index.html og.png CNAME .nojekyll README.md RUNBOOK.md site.manifest.yaml
git commit -m "Seed landing page"
gh repo create cjhorn95/establishedbefore.com --public --source . --remote origin --push
```
Verify the page is the approved one before publishing anything:
```bash
sha256sum index.html   # must print b372c9f505421f0ad340e41fca373e159ec3118574ebdb99a7c9e2b0eb4fb34e
```

### 3.2 GitHub Pages
```bash
gh api -X POST repos/cjhorn95/establishedbefore.com/pages -f 'source[branch]=main' -f 'source[path]=/'
gh api -X PUT  repos/cjhorn95/establishedbefore.com/pages -f cname=establishedbefore.com
gh api repos/cjhorn95/establishedbefore.com/pages --jq '{status,cname,https_enforced,cert:.https_certificate.state}'
```
The custom domain is also picked up automatically from the `CNAME` file on first build.
On Windows Git Bash prefix `gh api` calls with `MSYS_NO_PATHCONV=1`.

### 3.3 DNS (GoDaddy, zone `establishedbefore.com`, nameservers ns47/ns48.domaincontrol.com)
Set exactly this; leave MX, TXT (SPF/DKIM/DMARC), NS and SOA untouched — mail runs on this zone.

| Type | Host | Value | TTL |
|---|---|---|---|
| A | @ | 185.199.108.153 | 600 |
| A | @ | 185.199.109.153 | 600 |
| A | @ | 185.199.110.153 | 600 |
| A | @ | 185.199.111.153 | 600 |
| CNAME | www | cjhorn95.github.io | 600 |

Remove any parking A records (GoDaddy parking is `3.33.130.190` / `15.197.148.33`) and any
`www` CNAME that points at the apex.

`establishedbefore.co` and `establishedbefore.shop`: GoDaddy domain forwarding, 301
(permanent), destination `https://establishedbefore.com`, forward-only (no masking).

### 3.4 HTTPS
GitHub requests a Let's Encrypt cert once the apex A records resolve to GitHub (minutes to
about an hour). Re-asserting the domain nudges the check. When the cert state is `approved`:
```bash
gh api -X PUT repos/cjhorn95/establishedbefore.com/pages -F https_enforced=true
```
Until then the PUT returns 404 "The certificate does not exist yet" — wait, do not retry in a loop.

### 3.5 Updating the page later
Commit the newly approved `index.html` (and `og.png` if changed) to `main`, push.
commit and sha256 in §1 and in `site.manifest.yaml` (bump its `version`).

## 4. Verification

```bash
# run on Linux/Git Bash (dig, curl, sha256sum) — not PowerShell
dig +short A establishedbefore.com @8.8.8.8            # the four 185.199.108-111.153 addresses
dig +short CNAME www.establishedbefore.com @8.8.8.8    # cjhorn95.github.io.
curl -sI https://establishedbefore.com/ | head -1      # HTTP/2 200
curl -sI http://establishedbefore.com/  | head -1      # 301 once HTTPS is enforced
curl -s  https://establishedbefore.com/ | sha256sum    # b372c9f505421f0ad340e41fca373e159ec3118574ebdb99a7c9e2b0eb4fb34e
curl -sI https://establishedbefore.com/og.png | head -1 # 200
curl -sI https://establishedbefore.co/   | grep -i location   # https://establishedbefore.com
curl -sI https://establishedbefore.shop/ | grep -i location   # https://establishedbefore.com
gh api repos/cjhorn95/establishedbefore.com/pages --jq '.https_enforced'   # true
```
Before DNS has moved, test GitHub's copy directly:
`curl -s --resolve establishedbefore.com:80:185.199.108.153 http://establishedbefore.com/ | sha256sum`.

## 5. Access, by role (no secrets here)

| Surface | How | Who |
|---|---|---|
| GitHub repo + Pages | `gh` CLI as the `cjhorn95` account | GitHub / repo lane |
| GoDaddy DNS + forwarding | GoDaddy account, Google SSO | DNS / custody lane |
| Page design and copy | private fleet design repo, branch per change; the venture owner signs off | design lane briefs, venture owner approves |
| Publishing anything public | requires the operator's explicit per-change approval before the action | operator |

Account per role → `MemoryVault/Projects/_agents/gus/AI-SUBSCRIPTIONS-REGISTER.md` (private, fleet-internal). Nothing account-specific is kept in this repo.

## 6. History

| Date | Change |
|---|---|
| 2026-09-16 | Repo created; seeded with the interim landing page; Pages enabled (main, root); custom domain set. |
| 2026-09-16 | Page replaced with the approved chalkboard version (commit 1914629). |
| 2026-09-16 | DNS cut over from GoDaddy parking to GitHub Pages; `.co`/`.shop` forwarding set; HTTPS enforced when cert issued. |
| 2026-09-16 | Custom domain removed/re-added via the Pages API to force the cert request (GitHub wrote `12b73fa` Delete CNAME / `4880907` Create CNAME on main); Let's Encrypt cert issued 22:47 UTC; `https_enforced=true`. |
