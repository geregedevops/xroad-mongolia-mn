# cs.gerege.mn — operational history

A diary of every incident that touched the Central Server, what it broke, what the fix was, and what the next operator should look out for.

## 2026-04 — Initial install (xroad-center 7.8.0 on Ubuntu 24.04)

- NIIS upstream Debian packages installed: `xroad-centralserver`, `xroad-database-local`, `xroad-nginx`, `xroad-confclient`, `xroad-signer`, plus the management/registration services (`xroad-center-management-service`, `xroad-center-registration-service`).
- Postgres + xroad UI bootstrap done by the package post-install scripts. UI exposed on `4000/tcp` localhost only — operators access via `ssh -L 14000:localhost:4000 cs.gerege.mn`.
- Initial admin user `xrdadmin` created interactively during install.

**Watch out:** The package install opens an internal nginx on `4001/tcp` (globalconf) and `4002/tcp` (management service backend). It does NOT add UFW rules for those ports — every member SS that needs them has to be allow-listed explicitly. See incident below.

## 2026-04-19 — TLS handshake failed when rp.gerege.mn tried to register

**Symptom:** rp.gerege.mn produced `error_code.core.tls.handshake_failed` for every clientReg attempt, even though both ends had valid certs and the AUTH cert was registered + active.

**Root causes (multi-layer, peeled one at a time):**

1. **UFW blocked `4002/tcp` from rp IP.** Fix: `sudo ufw allow from 38.180.251.163 to any port 4002 proto tcp` (and the same for ss.gerege.mn 66.181.175.134 once it came online).
2. **OCSP responses (gerege-ocsp container) had drifted to ~3.5h old** while the X-Road freshness window was 3600s. Symptom propagated as `incorrect_validation_info: OCSP response is too old` inside signer.log. Fix: `docker restart gerege-ocsp` on gerege.mn → `systemctl restart xroad-signer` on rp.gerege.mn.
3. **The user's certs had `authorityInfoAccess = OCSP;URI:https://ocsp.gerege.mn` (no `/ocsp` path)**, so X-Road clients posted to root and were 405'd. Fixed by patching `gerege.mn` nginx (`ocsp.gerege.mn.conf`) to rewrite `POST /` → `/ocsp`. Also fixed `xroad-extensions.cnf` to use `https://ocsp.gerege.mn/ocsp` for any cert issued going forward.
4. **TSA cert in shared-params didn't match the live TSA signer.** The original install had a self-signed `TimeServer.mn Root CA` chain; we rebuilt the TSA chain to root in Gerege Root CA, but CS still had the old cert. Fix: CS UI → Trust Services → Timestamping Services → delete + re-add with the new leaf cert. See `timeserver.mn/HISTORY.md` for the full chain rebuild story.

**Watch out:** Whenever a member SS reports TLS handshake failure on registration, walk all four checks in order before assuming the SS itself is broken — the failure surfaces at the SS but the cause is almost always upstream (UFW, OCSP freshness, AIA URL, TSA cert mismatch).

## 2026-04-19 — TSA cert update via UI created a hash mismatch with `openssl x509 -fingerprint`

When you delete + re-add a TSA in CS UI, the cert is stored in `shared-params.xml` as base64 of the *PEM file text* (with `-----BEGIN CERTIFICATE-----` lines), not base64 of the raw DER. So `python -c "hashlib.sha256(base64.b64decode(<the xml field>)).hexdigest()"` will not equal `openssl x509 -fingerprint -sha256` of the same cert. Just decode as PEM (the bytes start with `-----BEGIN`) and you'll see the right cert. Spent 20 minutes on this thinking the upload had failed.

## 2026-04-20 — Pre-prod showcase exposed port 4000/tcp to public Internet

**Change:** `ufw allow 4000/tcp comment 'pre-prod showcase 2026-04-20'` on every MN host (cs, mgmt, ss, rp). The CS admin UI at `https://cs.gerege.mn:4000` is now reachable from any browser instead of the previous `ssh -L 14000:localhost:4000` tunnel-only access.

**Why:** showcase/demo convenience — screenshare of the UI without setting up tunnels for observers.

**Risk:** The CS UI is protected only by form-login (user `xrdadmin`). No mTLS, no IP allow-list, no WAF. A leaked admin password is fully sufficient for anyone on the Internet to impersonate `xrdadmin` and, from there, edit globalconf, revoke members, rotate the CS signing key, etc.

**Reverted 2026-04-22** across all four MN hosts (`sudo ufw delete allow 4000/tcp` on cs, mgmt, ss, rp; both IPv4 and IPv6 rules removed). Verified `ufw status` shows no `4000/tcp` entry on any host. Baseline access is back to `ssh -L 14000:localhost:4000 cs.gerege.mn`.

## 2026-04-22 — State-verification snapshot (taken for monorepo audit)

- **Running services match install-time expectation**: `xroad-center`, `xroad-center-management-service`, `xroad-center-registration-service`, `xroad-confclient`, `xroad-signer`, `xroad-nginx`, `postgresql@16-main`. No degraded/failed units. Uptime 4d 8h (boot 2026-04-18 UTC).
- **Signer softtoken holds two active p12s** (`74A8…4748.p12` added 2026-04-19 05:42, `D6CF…5790.p12` added 2026-04-19 05:43) alongside `.softtoken.p12`, plus a `keyconf.xml.predelete` sidecar from the same window — fallout from the TSA cert-rotation UI work. No action needed; kept for audit trail.
- **Listening ports**: `80` + `443` + `4001` public (nginx globalconf), `4002` restricted per-member (mgmt/rp/ss IPs), `4000` public (see showcase entry above), `8084`/`8085` + `5559`/`5560`/`5675` loopback-only (jetty + java IPC), `5432` postgres loopback.
- **Globalconf on disk refreshes every minute** (`/etc/xroad/globalconf/MN/shared-params.xml.metadata` mtime `2026-04-22 11:25`) — confclient loop healthy.

## Watch list for the next operator

- **GPG backup keyid in `local.ini`** — the value is REDACTED in this repo. Real keyid is in `reference_cs_secrets.md`. Rotating GPG breaks `xroad-confclient` resigning unless the new key is imported into `/etc/xroad/gpghome` first.
- **Postgres backups** are GPG-encrypted by `xroad-center` daily. They land in `/var/lib/xroad/backup/` and are pruned automatically. If you change the GPG key without keeping the old one for decryption you can't restore.
- **`managementService` URL inside `private-params.xml`** is `https://cs.gerege.mn:4001/managementservice/` (NOT 4002). Don't change to 4002 even though that's where the `manage/` services WSDL points — these are different code paths. 4001 is for auth-cert registration, 4002 is for client management ops.
- **`shared-params.xml` is regenerated and re-signed on every UI change.** Editing the file by hand on disk silently breaks confclient because the signature won't match. Always go through the UI.
