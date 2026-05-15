# Mongolian X-Road (instance `MN`)

Monorepo of every server, every config and every script that brings up the X-Road ecosystem operated by **Gerege Systems LLC** in Mongolia.

The instance identifier is **`MN`**. The Central Server lives at **cs.xroad.mn** and is currently the only authoritative source of `globalconf` for any Mongolian-X-Road-aware Security Server.

## Topology at a glance

```mermaid
graph TB
    %% Mongolia X-Road (instance MN) — high-level topology
    %% Solid arrows = X-Road message / control-plane paths
    %% Dotted arrows = attestation services (OCSP, CRL, RFC 3161 timestamps)

    subgraph trust[Trust services]
        direction LR
        CA["gerege.mn<br/>Root + Issuing CA<br/>OCSP / CRL / sign portal"]
        TSA["timeserver.mn<br/>RFC 3161 TSA<br/>(Sigstore, Gerege-rooted)"]
    end

    subgraph control[Instance control plane]
        direction LR
        CS["cs.xroad.mn<br/>Central Server (MN)<br/>signs globalconf"]
        MGMT["mgmt.xroad.mn<br/>Management SS<br/>publishes mgmt WSDL"]
    end

    subgraph members[Member security servers]
        direction LR
        RP["rp.gerege.mn<br/>Producer SS<br/>GEREGE-ID, EIDMONGOL"]
        SSG["ss.gerege.mn<br/>Consumer SS<br/>GEREGE-WALLET-BFF"]
        PAY["ss.paygrid.mn<br/>Member SS<br/>PAYGRID-CORE"]
    end

    IS["ca.gerege.mn /xroad/v1/*<br/>(IS behind GEREGE-ID,<br/>eid-gerege-backend)"]
    DEMO["test.gerege.mn<br/>(demo consumer, separate repo)"]

    CS -->|"globalconf 4001"| RP
    CS -->|"globalconf 4001"| SSG
    CS -->|"globalconf 4001"| PAY
    CS -->|"globalconf 4001"| MGMT
    MGMT -->|"mgmt proxy 4002"| CS

    SSG -->|"SS-SS 5500"| RP
    PAY -->|"SS-SS 5500"| RP
    RP -->|"HTTPS IS call"| IS
    DEMO -->|"REST :80"| SSG

    CA -.->|"OCSP / CRL"| RP
    CA -.->|"OCSP / CRL"| SSG
    CA -.->|"OCSP / CRL"| MGMT
    CA -.->|"OCSP / CRL"| PAY
    TSA -.->|"TSP timestamp"| RP
    TSA -.->|"TSP timestamp"| SSG
    TSA -.->|"TSP timestamp"| MGMT
    TSA -.->|"TSP timestamp"| PAY
```

Membership detail (member class, code, registered subsystems) lives in [`docs/topology.md`](docs/topology.md).

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
├── timeserver.mn/             RFC 3161 timestamping authority (Sigstore TSA, Gerege-rooted)
└── x-road.mn/                 Public landing page for the MN instance — static HTML +
                               precompiled Tailwind + Mermaid; live at https://x-road.mn
                               (38.180.242.76, same edge box as test.gerege.mn). Markets
                               the platform to potential member orgs and points back at
                               docs/ for the technical truth.
```

Each per-server folder has its own `README.md` describing the role, the ports it listens on, what files in `xroad/`, `nginx/`, `systemd/`, `tsa-certs/` etc. mean, and what to be careful about.

## Public IPs

| Host                | IP             | Role                                                             |
|---------------------|----------------|------------------------------------------------------------------|
| `cs.xroad.mn`       | 38.180.203.234 | X-Road Central Server (owner: Үндэсний дата төв since 2026-05-11; ops by Gerege Systems LLC) |
| `mgmt.xroad.mn`     | 38.180.255.177 | Management SS (owner: GOV/6806252, Цахим хөгжил инновац ЯЯ)        |
| `rp.gerege.mn`      | 38.180.251.163 | Producer SS (GEREGE-ID services)                                 |
| `ss.gerege.mn`      | 66.181.175.134 | Consumer SS (TEST-DEMO + future Gerege Core consumers)           |
| `ss.paygrid.mn`     | 38.180.254.231 | Member SS for Gerege Smart Metering / paygrid.mn (REGISTERED 2026-05-06)|
| `gerege.mn`         | 38.180.136.97  | Gerege Root CA, Issuing CA, OCSP, CRL, sign portal, /xroad/v1 IS |
| `timeserver.mn`     | 38.180.203.29  | TSA leaf signed by Gerege Root CA                                |

## Member identity overview

```mermaid
graph TB
    CS["MN (instance)"]

    GSY["MN/COM/6235972<br/>Gerege Systems LLC"]
    GCO["MN/COM/6884857<br/>Gerege Core LLC"]
    GSM["MN/COM/7181609<br/>Gerege Smart Metering"]
    GOV["MN/GOV/6806252<br/>Цахим хөгжлийн яам"]

    GID["GEREGE-ID<br/>producer"]
    GWEB["GEREGE-WEB<br/>producer"]
    EID["EIDMONGOL<br/>producer (e-ID v2)"]
    WBFF["GEREGE-WALLET-BFF<br/>consumer"]
    PCORE["PAYGRID-CORE<br/>hybrid"]
    MGMT_SUB["MANAGEMENT<br/>mgmt-svc"]

    CS --> GSY
    CS --> GCO
    CS --> GSM
    CS --> GOV
    GSY --> GID
    GSY --> GWEB
    GSY --> EID
    GCO --> WBFF
    GSM --> PCORE
    GOV --> MGMT_SUB

    classDef member fill:#E3F2FD
    classDef sub fill:#FFF8E1
    class GSY,GCO,GSM,GOV member
    class GID,GWEB,EID,WBFF,PCORE,MGMT_SUB sub
```

## Doc map

```mermaid
mindmap
  root((Mongolia X-Road MN<br/>documentation))
    Presentation
      docs/taniltsuulga.md
    Cross-cutting
      docs/topology.md
      docs/pki-architecture.md
      docs/onboarding-new-member-ss.md
      docs/operational-gotchas.md
      docs/mobile-security-roadmap.md
    Guide series
      docs/guides/architecture.md
      docs/guides/install-central-server.md
      docs/guides/install-security-server.md
      docs/guides/operate-central-server.md
      docs/guides/operate-security-server.md
      docs/guides/use-cases.md
      docs/guides/troubleshooting.md
      docs/guides/security.md
    Per-host
      cs.xroad.mn/
      mgmt.xroad.mn/
      rp.gerege.mn/
      ss.gerege.mn/
      ss.paygrid.mn/
      ca.gerege.mn/
      timeserver.mn/
    Public site
      x-road.mn/
```

Шинэ танилцагч: [`docs/taniltsuulga.md`](docs/taniltsuulga.md)-аас эхэл. Архитектор: [`docs/guides/architecture.md`](docs/guides/architecture.md). Алдаа гарвал: [`docs/guides/troubleshooting.md`](docs/guides/troubleshooting.md).

## Things this repo intentionally does NOT contain

- Private keys (CA root, CA issuing, TSA leaf, SS auth/sign keys, GPG backup keys).
- Database passwords, X-Road UI passwords, HSM PINs, FCM service-account JSONs.
- The literal value of `XROAD_SS_TOKEN` (the shared secret between rp.gerege.mn nginx and the gerege backend) — only the env var name and where it gets set.
- API tokens for `[management-service]` / `[registration-service]` in CS `local.ini`.

The operator's local memory store (under `~/.claude/.../memory/reference_cs_secrets.md`) records *where* each secret lives so it can be retrieved with `ssh + sudo` when needed.

## Sister repos

- [`gerege-mn-public`](https://github.com/geregedevops/gerege-mn-public) — public landing pages (e-id.mn, x-road.mn, template.gerege.mn), the full-stack eID/X-Road demo (test.gerege.mn), and the `gerege-doc-toolkit/` Markdown → branded `.docx` document toolkit used to render every PDF/Word artefact this monorepo produces.
