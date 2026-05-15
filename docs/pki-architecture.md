# PKI architecture

The Mongolia X-Road instance is rooted in a single trust anchor (`Gerege Root CA`) operated by Gerege Systems LLC on `gerege.mn`. No external trust roots are needed by SSes — the CS distributes only Gerege-issued CA certs in `shared-params.xml`.

## Hierarchy

```mermaid
graph TB
    ROOT["Gerege Root CA<br/>(self-signed, EC P-384)"]
    ISSUE["Gerege Issuing CA<br/>KU: keyCertSign + CRLSign<br/>no EKU restriction"]
    TSAISSUE["Gerege TSA Issuing CA<br/>CA:TRUE pathlen:0<br/>EKU critical: timeStamping"]

    XSS["X-Road auth/sign certs per SS<br/>(xroad_auth, xroad_sign profiles)"]
    INFRA["User AUTH/SIGN +<br/>OCSP responder +<br/>infrastructure certs"]
    TSALEAF["TimeServer.mn TSA Signer<br/>(leaf, EC P-256)<br/>KU crit digitalSignature<br/>EKU crit timeStamping"]

    ROOT --> ISSUE
    ROOT --> TSAISSUE
    ISSUE --> XSS
    ISSUE --> INFRA
    TSAISSUE --> TSALEAF
```

### Per-SS certificate issuance flow

```mermaid
sequenceDiagram
    autonumber
    participant SS as Partner SS UI<br/>(Keys & Certificates)
    actor OP as Operator on gerege.mn
    participant CA as openssl + Issuing CA key
    participant XR as Partner SS keyconf.xml

    SS->>SS: Generate key (xroad_auth / xroad_sign profile)
    SS->>OP: CSR (auth-csr.pem or sign-csr.pem)
    OP->>CA: sign-xroad-csr.sh <csr> auth|sign
    CA->>CA: openssl x509 -req<br/>-extfile xroad-extensions.cnf<br/>-extensions xroad_auth|xroad_sign
    CA-->>OP: signed .cer
    OP-->>SS: send .cer back
    SS->>XR: Import certificate → Activate
    Note over XR: cert state: registered → active<br/>presented on every X-Road handshake
```

## Per-cert profile in `xroad-extensions.cnf`

| Section          | Used for                          | KU                               | EKU                              | basicConstraints     |
|------------------|-----------------------------------|----------------------------------|----------------------------------|----------------------|
| `xroad_auth`     | SS authentication cert            | digitalSignature, keyEncipherment| clientAuth, serverAuth           | CA:FALSE             |
| `xroad_sign`     | SS message-signing cert           | nonRepudiation                   | emailProtection                  | CA:FALSE             |
| `xroad_tsa`      | TSA leaf                          | digitalSignature (critical)      | timeStamping (critical)          | CA:FALSE             |
| `tsa_issuing_ca` | Gerege TSA Issuing CA             | keyCertSign, CRLSign (critical)  | timeStamping (critical)          | CA:TRUE, pathlen:0   |

All certs include CRL distribution + AIA pointing to `https://crl.gerege.mn/issuing-ca.crl` and `https://ocsp.gerege.mn/ocsp`.

## Key storage

| Key                                  | Where it lives                                                                                  |
|--------------------------------------|--------------------------------------------------------------------------------------------------|
| Gerege Root CA private key           | gerege.mn `/opt/gerege-mn-eid/eid-gerege-backend/config/pki/root-ca.key`                         |
| Gerege Issuing CA private key        | gerege.mn `/opt/gerege-mn-eid/eid-gerege-backend/config/pki/issuing-ca.key`                      |
| Gerege TSA Issuing CA private key    | gerege.mn `/opt/xroad-ca/tsa-issuing/tsa-issuing.key`                                            |
| OCSP responder private key           | gerege.mn `/opt/gerege-mn-eid/eid-gerege-backend/config/pki/ocsp-responder.key`                  |
| TimeServer.mn TSA Signer private key | timeserver.mn `/opt/tsa-certs/leaf-key.pem`                                                      |
| Per-SS auth + sign keys              | The owning SS's `keyconf.xml` (managed by xroad-signer; never leaves the SS)                    |
| User AUTH + SIGN keys                | Inside the user's Gerege ID app on their phone (HSM-backed where available)                      |

PKI hardening status (SoftHSM2 backbone, autobackup cron, leading-zero serial fix) is tracked in the operator's local memory. The EC-HSM code refactor in gerege-ocsp/backend remains as future work.
