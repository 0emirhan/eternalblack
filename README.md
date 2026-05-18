<div align="center">

# EternalBlack

**`eternalblack.fr`**

*Fictional breach forum & cybercrime marketplace — public OSINT training simulation.*

</div>

---

## What this is

[`eternalblack.fr`](https://eternalblack.fr) is a fully fabricated cybercrime
forum / breach-forum / marketplace stood up as a **permanent target range**
for OSINT collectors and pivot engines.

Every name, wallet, key, document, dump, paste, combolist, config, profile
and page on the site is **fake**. The site exists so that automated
investigation tooling can be exercised against a stable, dense,
cross-referenced target without touching real infrastructure.

It is a test module for **OpenCrime** and **osint-engine**.

---

## ⚠ This is a simulation

Nothing on EternalBlack is real:

- Identities, pseudos, biographies, photos — fabricated
- Email addresses (Proton, Tuta, cock.li, etc.) — fabricated
- BTC / ETH / Monero wallet addresses — non-functional lookalikes
- PGP fingerprints, keys and ASCII-armored blocks — placeholders
- SSH keys, JWT tokens, API keys, AWS credentials — dummies
- Database dumps, combolists, paste content — generated nonsense
- Shell companies, registration numbers, WHOIS contacts — invented
- On-chain transactions, blocks, hashes — synthesized
- PDF metadata (Author / Creator / Producer / dates) — spoofed
- HTTP headers (`Server`, `X-Powered-By`, `X-Forum-Version`) — spoofed
- Analytics IDs (GA, GTM, FB pixel, Matomo, Hotjar, Yandex) — fake

Disclosure is present at every layer: red bar at the top of every page,
dedicated `/about` page, every PDF letterhead, every leak file header,
`/.well-known/security.txt`, `/humans.txt`, and an `X-Simulation` HTTP
response header.

---

## What you can find on the site

The public surface is intentionally minimal: landing on `/` shows only a
login gate and a "members only" notice. The actual content lives behind
**obscured deep paths** that your tooling has to discover the way it would
on a real underground site.

### Public surface

| Path | What it serves |
|---|---|
| `/` | Login gate, members-only notice |
| `/forum` | 2 pinned public threads + N gated threads notice |
| `/forum/thread/{id}` | Pinned threads readable; gated threads show a notice |
| `/rules` | Forum rules (cosmetic) |
| `/contact` | Staff contact block |
| `/about` | Explicit simulation disclosure |
| `/login`, `/register` | Auth forms |

### Members & marketplace (hidden)

| Path | What it exposes |
|---|---|
| `/panel/roster` | Member directory (15 vetted members) |
| `/panel/member/{id}` | Full profile: rank, credits, bio, contact, wallets, IPs history, devices, vouches, documents, threads, pastes |
| `/panel/search?q=` | Cross-index search (users / listings / threads / pastes) |
| `/m/board` | Marketplace (13 listings) |
| `/m/listing/{id}` | Listing detail with seller IOC block |

### Intel feed (breach-forum content, hidden)

| Path | What it exposes |
|---|---|
| `/intel/feed` | Aggregate leaks + pastes index |
| `/intel/databases` | **15 leaked databases** (free + paid tiers) |
| `/intel/database/{id}` | DB detail with download link |
| `/intel/combolists` | **7 combolists** (`email:pass` / `user:pass`) |
| `/intel/combolist/{id}` | Combolist detail |
| `/intel/cracking` | **5 cracking configs** (OpenBullet 2 / SilverBullet) |
| `/intel/config/{id}` | Config detail |
| `/intel/tutorials` | **6 how-to tutorials** (hashcat, OpenBullet, BPH, Monero opsec…) |
| `/intel/tutorial/{id}` | Tutorial body |
| `/intel/paste/{id}` | Paste content (env leaks, JWT samples, PoC stubs, dumps…) |
| `/intel/breach/{id}` | Historical breach detail |

### Entities, ledger, archives, downloads (hidden)

| Path | What it exposes |
|---|---|
| `/entities/registry` | **9 shell companies** with reg number, jurisdiction, directors, WHOIS-flavored emails |
| `/chain/ledger` | **8 on-chain transactions** chained to form a traceable money flow |
| `/cdn/archive` | **15 PDF documents** (invoices, KYC, SLAs, audits, whitepapers, manuals, CVE drafts…) |
| `/cdn/archive/{filename}` | PDF download with **spoofed metadata** (Author, Creator, Producer, dates with jurisdiction-correct timezones) |
| `/cdn/leaks/{filename}` | **27 downloadable leak files** (.sql / .csv / .txt / .loli / .svb) |
| `/cdn/pgp/{id}.asc` | Per-user PGP public key block |
| `/backup/db` | SQLite + per-table CSV/JSON/XML exports (19 tables) |
| `/admin` | Weak-auth admin panel (`admin` / `admin123`) |
| `/admin/dashboard` | All users, listings, breaches |
| `/admin/debug` | phpinfo-style env leak + complete internal route map |

### Discovery surface (intentionally exposed)

| Path | Why it's there |
|---|---|
| `/robots.txt` | Lists every hidden path as a `Disallow:` entry |
| `/sitemap.xml` | Enumerates every deep URL on the site |
| `/.well-known/security.txt` | Simulation disclosure + maintainer |
| `/humans.txt` | Simulation disclosure |
| `/.env` | Fake DB / Redis / S3 / JWT secrets + internal `ROUTE_*` map |
| `/.git/config`, `/.git/HEAD`, `/.git/logs/HEAD` | Fake repo metadata + commit authors |
| `/backup.sql` | Partial nightly user-table dump |
| `/crossdomain.xml` | Overly permissive Flash policy |

### API surface

| Path | What it returns |
|---|---|
| `/api/v1/` | Endpoint index |
| `/api/v1/docs` | Swagger UI (OpenAPI auto-generated) |
| `/api/v1/users`, `/api/v1/users/{id}`, `/api/v1/users/{id}/extra` | Full user data + extended profile (IPs, devices, previous handles) |
| `/api/v1/users/{id}/timeline`, `/api/v1/users/{id}/devices`, `/api/v1/users/{id}/ips` | Per-user views |
| `/api/v1/listings`, `/api/v1/threads`, `/api/v1/pastes`, `/api/v1/breaches`, `/api/v1/companies` | Listings / forum / leaks / entities |
| `/api/v1/transactions?wallet={addr}` | On-chain ledger filtered by wallet |
| `/api/v1/pivot?type={email\|wallet\|alias\|company}&q=` | Unified pivot endpoint |
| `/api/v1/pivot/chain?start_user_id=` | Multi-hop graph walk |
| `/api/v1/messages?user_id=` | **IDOR by design** — any user_id returns "private" messages |
| `/api/v1/chats/{id}` | **IDOR by design** — any chat_id returns full history |
| `/api/v1/config` | **Secrets leak by design** + internal route map |
| `/api/v1/admin/logs` | **Unauth log stream by design** |
| `/api/v1/admin/users/export` | **Unauth full user dump by design** |
| `/api/v1/vouches`, `/api/v1/replies?thread_id=`, `/api/v1/reviews?listing_id=` | Vouches / replies / reviews |
| `/api/bd/` | DB export index |
| `/api/bd/eternalblack.sqlite` | Full SQLite database (19 tables) |
| `/api/bd/{table}.csv\|.json\|.xml` | Per-table exports |

---

## OSINT pivot surface

| Pivot type | Where it surfaces on the site |
|---|---|
| Alias / handle (Telegram, Jabber, Keybase, GitHub, GitLab, Twitter, Reddit, Signal, Matrix, Mastodon, Medium, LinkedIn, Stack Overflow, IRC, Discord) | `/panel/member/*`, threads, pastes, chats, `aliases` table |
| Previous handle | `/panel/member/*`, `previous_usernames` table |
| Email | `/panel/member/*`, pastes, breaches, `/backup.sql`, `/api/v1/pivot` |
| BTC / ETH / XMR wallet | `/panel/member/*`, `/m/listing/*`, `/chain/ledger`, `wallets` table |
| PGP fingerprint | `/panel/member/*`, `/cdn/pgp/{id}.asc`, pastes |
| SSH key | Selected users, pastes, `/.env` |
| HTTP server fingerprint | `Server`, `X-Powered-By`, `X-Forum-Version` headers |
| Favicon hash | `/static/img/favicon.svg` |
| Analytics ID | Meta tags on every page |
| IP (historic) | `/panel/member/*`, `/.env`, `/api/v1/config`, `/backup.sql`, `known_ips` table |
| Device fingerprint + UA | `/panel/member/*`, `/api/v1/users/{id}/devices` |
| WHOIS email | `/entities/registry` |
| Domain | `/entities/registry`, pastes, threads |
| Shell company | `/entities/registry`, member profiles, documents |
| CVE / exploit reference | Threads, pastes, PDFs |
| Secrets (env) | `/.env`, `/api/v1/config`, `/admin/debug` |
| Git metadata | `/.git/*` |
| PDF metadata (Author / Creator / Producer / dates / TZ) | `/cdn/archive/*.pdf` |
| Leaked DB content (sample rows extractable) | `/cdn/leaks/*.sql`, `/cdn/leaks/*.csv` |
| Combolist | `/cdn/leaks/combo_*.txt` |
| Cracking config | `/cdn/leaks/*.loli`, `/cdn/leaks/*.svb` |
| Chat messages | `/api/v1/chats/{id}` |

---

## Inventory in numbers

- **15** vetted members with rank (God / Administrator / Staff / Vendor /
  VIP / Member / Leecher / Banned) and credit balances
- **9** shell companies with directors and WHOIS-flavored emails
- **13** marketplace listings with reviews
- **31** forum threads across **12** sections
- **15** leaked databases (free + paid tiers)
- **7** combolists
- **5** OpenBullet / SilverBullet cracking configs
- **6** how-to tutorials
- **8** historical pastes
- **6** historical breaches
- **8** on-chain transactions
- **5** chat rooms with fake messages
- **9** vouches
- **15** PDF documents with spoofed metadata
- **27** downloadable leak files (`.sql` / `.csv` / `.txt` / `.loli` / `.svb`)
- **19** SQLite tables in the full DB export

---

## Maintainer

[github.com/0emirhan](https://github.com/0emirhan)
