# ca.gerege.mn — PKI + OCSP + CRL + sign portal + GEREGE-ID Information System

**Public IP:** 38.180.136.97
**Hostname (uname):** `a602286395.local` (Cogent default; the host has no canonical FQDN, only the brand vhosts)
**Owner:** Gerege Systems LLC (`MN/COM/6235972`)
**Role:** The everything-PKI host. Runs:

- **Gerege Root CA** — self-signed root, signs the issuing CA. EC P-384, key on disk.
- **Gerege Issuing CA** — signs every X-Road auth/sign cert, every user AUTH/SIGN cert, OCSP responder cert, etc.
- **Gerege TSA Issuing CA** — separate intermediate signed by the Root, with `extendedKeyUsage = critical, timeStamping`. Only purpose: to issue the timeserver.mn TSA leaf so Sigstore TSA accepts the chain (Sigstore enforces timeStamping EKU on every non-root cert in `certchain.pem`).
- **OCSP responder** (`gerege-ocsp` container, `https://ocsp.gerege.mn`) — RFC 6960. Responds at `/ocsp` and at `/` (POST root rewrite to `/ocsp` for X-Road client compatibility).
- **CRL distribution** (`gerege-crl` container, `https://crl.gerege.mn`) — issues issuing-ca.crl.
- **Sign portal** (`gerege-sign` container, `https://sign.gerege.mn`) — UI for org admins to sign uploaded X-Road CSRs via the Issuing CA.
- **eid-gerege-backend** (Go/Fiber, `https://ca.gerege.mn`) — the user-facing /api + the X-Road Information System endpoints `/xroad/v1/*` that the GEREGE-ID producer SS forwards to.

## Folder map

```
ca.gerege.mn/
├── README.md (this file)
├── nginx/
│   ├── ca.gerege.mn.conf      — IS endpoint /xroad/v1 + OpenAPI3 at /xroad/openapi/
│   ├── ocsp.gerege.mn.conf    — root POST → /ocsp rewrite (X-Road client posts to root)
│   ├── crl.gerege.mn.conf     — CRL static serve
│   ├── sign.gerege.mn.conf    — sign portal frontend
│   └── gerege.mn.conf         — main brand site (in /etc/nginx/sites-available on the host but DISABLED — the symlink in sites-enabled was removed 2026-04-28 when the host stopped serving the brand site directly; preserved here as a reference for the SSL/security-headers stanza)
└── xroad-ca/
    ├── xroad-extensions.cnf   — openssl ext profiles: xroad_sign, xroad_auth, xroad_tsa, tsa_issuing_ca
    ├── sign-xroad-csr.sh      — wraps openssl x509 -req with the right extension; accepts sign|auth|tsa
    └── tsa-issuing/
        ├── tsa-issuing.csr    — CSR for Gerege TSA Issuing CA (cert below was signed from this)
        └── tsa-issuing.pem    — Gerege TSA Issuing CA cert (signed by Gerege Root)
```

Untracked vhosts on this host (intentionally not in repo, just noted so an operator doing `ls /etc/nginx/sites-enabled` is not surprised): `eid.gerege.mn` and `id.gerege.mn` are reverse proxies to `127.0.0.1:3000` (the e-ID portal Next.js app), enabled 2026-04-28. Both share the LE cert at `/etc/letsencrypt/live/id.gerege.mn/fullchain.pem`. They do not interact with `/xroad/v1/*` and have no IS-token gating.

## How `/xroad/v1` IS gating works

The gerege backend trusts the `X-Road-Client` header only when the request also carries `X-Gerege-SS-Token: <secret>` (set as `XROAD_SS_TOKEN` env var). `nginx/ca.gerege.mn.conf` is the gate: only requests where `$remote_addr == 38.180.251.163` (rp.gerege.mn) reach a `location /xroad/v1/` block that injects the header. Every other source IP that hits `/xroad/v1/` gets `403 X-Road IS endpoint restricted` from nginx; even if they could reach the backend port directly the middleware would still reject because they can't forge `X-Gerege-SS-Token`.

The literal token is in `/opt/gerege-mn-eid/eid-gerege-backend/.env` on this server (variable `XROAD_SS_TOKEN`). Replace it by regenerating with `openssl rand -hex 32`, updating both the `.env` value AND the matching `proxy_set_header X-Gerege-SS-Token "..."` line in `ca.gerege.mn.conf`, then `nginx -s reload && docker compose up -d backend`.

## Cert chain shape

```
Gerege Root CA (self-signed, EC P-384)
├── Gerege Issuing CA              (KU: keyCertSign+CRLSign; no EKU restriction)
│   ├── X-Road auth certs          (xroad_auth profile: digitalSignature+keyEncipherment, clientAuth+serverAuth)
│   ├── X-Road sign certs          (xroad_sign profile: nonRepudiation, emailProtection)
│   ├── User AUTH certs            (per Mongolian-citizen-on-Gerege-ID)
│   ├── User SIGN certs            (per Mongolian-citizen-on-Gerege-ID)
│   ├── OCSP responder cert
│   └── (other infrastructure certs)
└── Gerege TSA Issuing CA          (CA:TRUE pathlen:0, EKU: critical,timeStamping)
    └── TimeServer.mn TSA Signer   (xroad_tsa profile: critical digitalSignature, critical timeStamping)
```

Why the second intermediate exists: Sigstore TSA validates that every non-root cert in its `certchain.pem` carries `id-kp-timeStamping`. The general-purpose Gerege Issuing CA has no EKU restriction, so a TSA leaf signed under it would get rejected by Sigstore at startup with `panic: certificate must have extended key usage timestamping set`. Carving out a TSA-only intermediate keeps both the X-Road profile (no EKU on intermediates) and the Sigstore profile (EKU on every intermediate) happy.

## Operational gotchas

- **`gerege-ocsp` container caches responses** for `freshness * 0.7` seconds and a stale response will fail every X-Road OCSP verification with `incorrect_validation_info: OCSP response is too old`. `docker restart gerege-ocsp` then `systemctl restart xroad-signer` on the affected SS clears it.
- **OCSP cert AIA** in every issued cert must include the `/ocsp` path: `authorityInfoAccess = OCSP;URI:https://ocsp.gerege.mn/ocsp`. Without `/ocsp`, the X-Road client posts to `/` and historically that 405'd; nginx now rewrites root POST → `/ocsp` to keep older certs working too.
- **OCSP responder and the leading-zero serial bug** — the responder used to drop the leading zero byte from positive cert serials, breaking lookups for certs whose serial happened to start with 0x00. Patched 2026-04 and serials are now always reproduced byte-exact.
