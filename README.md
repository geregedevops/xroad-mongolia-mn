# Mongolian X-Road (instance `MN`)

Monorepo of every server, every config and every script that brings up the X-Road ecosystem operated by **Gerege Systems LLC** in Mongolia.

The instance identifier is **`MN`**. The Central Server lives at **cs.xroad.mn** and is currently the only authoritative source of `globalconf` for any Mongolian-X-Road-aware Security Server.

## Topology at a glance

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Mongolian X-Road (MN)                              │
│                                                                             │
│   gerege.mn ─────► CA + OCSP + CRL          timeserver.mn ─► RFC 3161 TSA   │
│   (root + issuing + tsa-issuing CA)          (Sigstore TSA, Gerege-rooted)  │
│                                                                             │
│   cs.xroad.mn  ─► X-Road Central Server (instance MN, signs globalconf)     │
│   mgmt.xroad.mn  ► Management SS (publishes management services WSDL)       │
│                                                                             │
│   ┌───────────────────────────┬───────────────────────────────────────┐     │
│   │ Member: Gerege Systems LLC│ Member: Gerege Core LLC               │     │
│   │ COM/6235972               │ COM/6884857                           │     │
│   │ rp.gerege.mn (producer SS)│ ss.gerege.mn (consumer SS)            │     │
│   │   └─ GEREGE-ID subsystem  │   └─ TEST-DEMO subsystem              │     │
│   │      auth-svc, sign-svc,  │                                       │     │
│   │      cert-svc REST OpenAPI│                                       │     │
│   ├───────────────────────────┴───────────────────────────────────────┤     │
│   │ Member: Gerege Smart Metering (COM/7181609)  ss.paygrid.mn        │     │
│   │   REGISTERED on CS 2026-05-06; PAYGRID-SS-1; brand domain         │     │
│   │   paygrid.mn; subsystem PAYGRID-CORE registered 2026-05-07.       │     │
│   └───────────────────────────────────────────────────────────────────┘     │
│                                                                             │
│   Information system behind GEREGE-ID:                                      │
│   ca.gerege.mn:443  ──► nginx (gerege.mn host) ──► /xroad/v1/* in           │
│                          eid-gerege-backend                                 │
│                                                                             │
│   Demo consumer of the whole flow: test.gerege.mn (separate repo).          │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Repo layout

```
mongolian-xroad-mn/
├── README.md                  ← this file
├── docs/
│   ├── topology.md            full host/IP/port/cert table + cert chain diagrams
│   ├── pki-architecture.md    Gerege Root → Issuing CA + TSA Issuing CA + per-cert profile
│   ├── onboarding-new-member-ss.md  end-to-end checklist for a partner SS
│   └── operational-gotchas.md OCSP staleness, TSP cert hash mismatch, cert URL-encode etc.
├── cs.xroad.mn/               Central Server (X-Road v7.8.0)
├── mgmt.xroad.mn/             Management Security Server (owned by Цахим хөгжил инновац харилцаа холбооны яам / GOV/6806252 since 2026-05-08; was Gerege Systems LLC)
├── rp.gerege.mn/              Producer SS publishing GEREGE-ID auth/sign/cert services
├── ss.gerege.mn/              Consumer SS owning the TEST-DEMO subsystem (Gerege Core LLC)
├── ss.paygrid.mn/             Member SS for Paygrid LLC (xroad-securityserver 7.8.0, wizard pending)
├── ca.gerege.mn/              CA + OCSP + CRL + sign portal + X-Road IS for GEREGE-ID
│                              (vhosts: gerege.mn, ca., ocsp., crl., sign. on 38.180.136.97)
└── timeserver.mn/             RFC 3161 timestamping authority (Sigstore TSA, Gerege-rooted)
```

Each per-server folder has its own `README.md` describing the role, the ports it listens on, what files in `xroad/`, `nginx/`, `systemd/`, `tsa-certs/` etc. mean, and what to be careful about.

## Public IPs

| Host                | IP             | Role                                                             |
|---------------------|----------------|------------------------------------------------------------------|
| `cs.xroad.mn`       | 38.180.203.234 | X-Road Central Server                                            |
| `mgmt.xroad.mn`     | 38.180.255.177 | Management SS (owner: GOV/6806252, Цахим хөгжил инновац ЯЯ)        |
| `rp.gerege.mn`      | 38.180.251.163 | Producer SS (GEREGE-ID services)                                 |
| `ss.gerege.mn`      | 66.181.175.134 | Consumer SS (TEST-DEMO + future Gerege Core consumers)           |
| `ss.paygrid.mn`     | 38.180.254.231 | Member SS for Gerege Smart Metering / paygrid.mn (REGISTERED 2026-05-06)|
| `gerege.mn`         | 38.180.136.97  | Gerege Root CA, Issuing CA, OCSP, CRL, sign portal, /xroad/v1 IS |
| `timeserver.mn`     | 38.180.203.29  | TSA leaf signed by Gerege Root CA                                |

## Things this repo intentionally does NOT contain

- Private keys (CA root, CA issuing, TSA leaf, SS auth/sign keys, GPG backup keys).
- Database passwords, X-Road UI passwords, HSM PINs, FCM service-account JSONs.
- The literal value of `XROAD_SS_TOKEN` (the shared secret between rp.gerege.mn nginx and the gerege backend) — only the env var name and where it gets set.
- API tokens for `[management-service]` / `[registration-service]` in CS `local.ini`.

The operator's local memory store (under `~/.claude/.../memory/reference_cs_secrets.md`) records *where* each secret lives so it can be retrieved with `ssh + sudo` when needed.

## Sister repos

- [`gerege-mn-public`](https://github.com/geregedevops/gerege-mn-public) — public landing pages (e-id.mn, x-road.mn, template.gerege.mn), the full-stack eID/X-Road demo (test.gerege.mn), and the `gerege-doc-toolkit/` Markdown → branded `.docx` document toolkit used to render every PDF/Word artefact this monorepo produces.
