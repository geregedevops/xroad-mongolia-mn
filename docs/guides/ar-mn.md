# АГ-MN — Mongolia X-Road архитектур (Architecture Guide)

> **Doc code:** AR-MN — analogous to NIIS `AR-CS`/`AR-SS`/`AR-EXT` сериал.
> **Зорилго:** Mongolia X-Road instance `MN`-ийн архитектурын **бүх давхрага** — конкреттай хост, демон, өгөгдлийн загвар, итгэлийн загвар хүртэл — нэг газар цуглуулсан reference document. Танилцуулга биш, **архитектор / SRE / security review хийгчдэд зориулсан**.
> **Урьдач уншлага:** [`docs/taniltsuulga.md`](../taniltsuulga.md) ерөнхий тойм, [`docs/topology.md`](../topology.md) хост хүснэгт.

---

## Агуулга

1. [Архитектурын зорилго, хязгаар](#1-архитектурын-зорилго-хязгаар)
2. [4+1 view-той тойм](#2-41-view-той-тойм)
3. [Логик архитектур](#3-логик-архитектур)
4. [Процесс / runtime архитектур](#4-процесс--runtime-архитектур)
5. [Deployment архитектур](#5-deployment-архитектур)
6. [Өгөгдлийн архитектур](#6-өгөгдлийн-архитектур)
7. [Аюулгүй байдлын архитектур](#7-аюулгүй-байдлын-архитектур)
8. [Сүлжээний архитектур](#8-сүлжээний-архитектур)
9. [Интеграцийн загвар](#9-интеграцийн-загвар)
10. [Архитектурын шийдвэрийн бүртгэл (ADR)](#10-архитектурын-шийдвэрийн-бүртгэл-adr)
11. [Гадаад хамаарал ба upstream](#11-гадаад-хамаарал-ба-upstream)

---

## 1. Архитектурын зорилго, хязгаар

### 1.1 Архитектурын зорилго

- **Trustworthy data exchange**: гишүүн байгууллагуудын IS хооронд гэрчилсэн, гарын үсэгтэй, цаг тэмдэглэлтэй, аудит логтой мессеж дамжуулах.
- **Decentralized но governed**: гишүүд бие даан үйл ажиллагаа явуулна; зөвхөн Central Server-ийн approved тохиргооны хүрээнд харилцана.
- **Compliance**: Mongolia E-Sign Law, ISO/IEC 27001 контролтой, GDPR-төстэй өгөгдлийн privacy.
- **Legally significant**: Mongolian-eID-тэй уялдан, цахим гарын үсэг бүхий зурвас хууль зүйн ач холбогдолтой.
- **Audit-ready**: бүх SS-SS мессежийг X-Road message log дотор гарын үсэг + timestamp-тай ор хадгалдаг (`xroad-addon-messagelog`).

### 1.2 Хүрээний хязгаар (scope boundary)

```mermaid
graph LR
    subgraph in_scope["In scope (this guide)"]
        CS[Central Server architecture]
        SS[Security Server architecture]
        PKI[PKI + OCSP + CRL]
        TSA[Time-Stamp Authority]
        MSG[Message envelope + transport]
        ACL[ACL / Service-clients model]
    end

    subgraph partial["Partial scope (referenced)"]
        IS[Information Systems (per member)]
        MOB[Mobile-side hardening]
        FED[Federation with other instances]
    end

    subgraph out_of_scope["Out of scope"]
        BIZ[Business logic in member apps]
        NET[ISP-level network engineering]
        LEGAL[Legal contract templates]
    end

    in_scope --> partial
    partial --> out_of_scope
```

### 1.3 Архитектурын зарчмууд

| Зарчим | Тайлбар |
|---|---|
| **Trust through cryptography, not network** | Public Internet дамждаг бүх SS-SS зурвас mTLS + cert-pinning-тэй. Зөвхөн зөв cert chain + Service-clients ACL хоёулангаар зөвшөөрөгдөнө. |
| **Single source of truth for membership** | Гишүүн identity (member class+code+subsystem) зөвхөн CS-ийн `shared-params.xml`-аас, өөр газар duplicate байхгүй. |
| **Every signing key isolated to a single host** | Root CA private гэвэл `ca.gerege.mn`. TSA leaf private гэвэл `timeserver.mn`. SS AUTH/SIGN private гэвэл тухайн SS-ийн `keyconf.xml`. Хэзээ ч ил гарахгүй. |
| **Time-bound trust** | OCSP fresh window 3600s, TSA token expiry, cert validity 825 хоног (TSA leaf), 2 жил+ (SS AUTH/SIGN) гэх мэт. |
| **No DB write per partner on producer-side** | Service-clients ACL = single source of truth. `eid-gerege-backend` зөвхөн `00000001-...` X-Road gateway row-той (2026-04-19 refactor, `ca.gerege.mn/HISTORY.md` 2026-04-19). |

---

## 2. 4+1 view-той тойм

Krchten 4+1 архитектурын моделийг ашиглаж олон үзэгдлээр харна:

```mermaid
mindmap
  root((Mongolia X-Road<br/>architecture views))
    Logical
      Components per host
      Subsystem identities
      Trust hierarchy
    Process
      xroad-proxy / signer / confclient
      OCSP responder cache
      TSA signing
    Deployment
      7 physical hosts
      Ubuntu 24.04 / Postgres 16
      Docker для CA stack
    Data
      shared-params.xml
      private-params.xml
      keyconf.xml
      serverconf DB
      messagelog DB
      certificates table
    Scenarios
      Member onboarding
      Service call
      Cert renewal
      Incident response
```

Дараагийн 6 хэсэгт тус бүрийг гүн авч үзнэ.

---

## 3. Логик архитектур

### 3.1 Гол компонентууд (UML-стайл class breakdown)

```mermaid
classDiagram
    class CentralServer {
        +instanceId: string = "MN"
        +signingKey: PrivateKey
        +members: Member[]
        +approvedCAs: CA[]
        +approvedTSAs: TSA[]
        +centralServices: Service[]
        +signGlobalconf()
        +approveManagementRequest(req)
    }

    class SecurityServer {
        +memberId: MemberId
        +serverCode: string
        +authCert: X509
        +signCert: X509
        +tsps: TSP[]
        +clients: Client[]
        +configAnchor: Anchor
        +confclient: ConfClient
        +signer: Signer
        +proxy: XRoadProxy
    }

    class Member {
        +instance: "MN"
        +memberClass: enum(COM,GOV,NEE)
        +memberCode: string
        +name: string
        +subsystems: Subsystem[]
    }

    class Subsystem {
        +code: string
        +status: enum(SAVED,REGISTERED)
        +internalServers: IS[]
        +services: Service[]
        +serviceClients: ACL[]
    }

    class Service {
        +code: string
        +url: string
        +openApiSpec: ref
    }

    class ACL {
        +subject: SubsystemId
        +operations: string[]
    }

    class CA {
        +cert: X509
        +ocspUrl: string
        +crlUrl: string
    }

    class TSA {
        +cert: X509
        +url: string
    }

    CentralServer "1" --> "*" Member
    CentralServer "1" --> "*" CA
    CentralServer "1" --> "*" TSA
    CentralServer "1" o-- "*" SecurityServer : registered
    Member "1" --> "*" Subsystem
    SecurityServer "1" --> "*" Subsystem : hosts
    Subsystem "1" --> "*" Service : publishes
    Service "1" --> "*" ACL
```

### 3.2 Mongolia дахь конкрет бичлэг (instance population)

```mermaid
graph TB
    CS["MN (instance)"]
    GSY["MN/COM/6235972<br/>Gerege Systems LLC"]
    GCO["MN/COM/6884857<br/>Gerege Core LLC"]
    GSM["MN/COM/7181609<br/>Gerege Smart Metering"]
    GOV["MN/GOV/6806252<br/>Цахим хөгжлийн яам"]
    OTH["MN/COM/6658679<br/>Gerege Edu (subsystem only)"]
    GREDU["MN/COM/6975291<br/>Tasker (subsystem only)"]

    GID["GEREGE-ID<br/>(producer)"]
    GWEB["GEREGE-WEB<br/>(producer)"]
    EID["EIDMONGOL<br/>(producer)"]
    WBFF["GEREGE-WALLET-BFF<br/>(consumer)"]
    PCORE["PAYGRID-CORE<br/>(member, hybrid)"]
    MGMT["MANAGEMENT<br/>(mgmt-svc)"]
    DBANK1["BANK1-DBANK"]
    DBANK2["BANK2-DBANK"]
    DBANK3["BANK3-DBANK"]
    NBFI1["NBFI1-DEMO"]
    NBFI2["NBFI2-DEMO"]

    CS --> GSY
    CS --> GCO
    CS --> GSM
    CS --> GOV

    GSY --> GID
    GSY --> GWEB
    GSY --> EID
    GCO --> WBFF
    GSM --> PCORE
    GOV --> MGMT
    GOV --> DBANK1
    GOV --> DBANK2
    GOV --> DBANK3
    GOV --> NBFI1
    GOV --> NBFI2

    classDef member fill:#E3F2FD
    classDef sub fill:#FFF8E1
    class GSY,GCO,GSM,GOV,OTH,GREDU member
    class GID,GWEB,EID,WBFF,PCORE,MGMT,DBANK1,DBANK2,DBANK3,NBFI1,NBFI2 sub
```

### 3.3 Service-clients ACL топологи

```mermaid
flowchart LR
    %% Service-clients = who can call what?

    subgraph providers["Producer subsystems"]
        GID_AUTH[GEREGE-ID/auth-svc]
        GID_SIGN[GEREGE-ID/sign-svc]
        GID_CERT[GEREGE-ID/cert-svc]
        EID_AUTH[EIDMONGOL/auth-svc]
        EID_SIGN[EIDMONGOL/sign-svc]
    end

    subgraph consumers["Consumer subsystems"]
        GW[GEREGE-WEB]
        TD[TEST-DEMO]
        CMN[CONTRACT-MN]
        WBFF[GEREGE-WALLET-BFF]
        GE[GEREGE-EDU]
        TASK[TASKER]
        DBANK1[BANK1-DBANK]
        DBANK2[BANK2-DBANK]
        DBANK3[BANK3-DBANK]
        PCORE[PAYGRID-CORE]
    end

    GW --> GID_AUTH
    GW --> GID_SIGN
    GW --> GID_CERT
    TD --> GID_AUTH
    TD --> GID_SIGN
    TD --> GID_CERT
    CMN --> GID_AUTH
    CMN --> GID_SIGN
    WBFF --> GID_AUTH
    WBFF --> GID_SIGN
    GE --> GID_AUTH
    GE --> GID_SIGN
    GE --> GID_CERT
    TASK --> GID_AUTH
    TASK --> GID_SIGN
    TASK --> GID_CERT

    CMN --> EID_AUTH
    CMN --> EID_SIGN
    WBFF --> EID_AUTH
    WBFF --> EID_SIGN
    GE --> EID_AUTH
    GE --> EID_SIGN
    TASK --> EID_AUTH
    TASK --> EID_SIGN
    DBANK1 --> EID_AUTH
    DBANK1 --> EID_SIGN
    DBANK2 --> EID_AUTH
    DBANK2 --> EID_SIGN
    DBANK3 --> EID_AUTH
    DBANK3 --> EID_SIGN
    PCORE --> EID_AUTH
    PCORE --> EID_SIGN
```

★ **Insight ─────────────────────────────────────**
- EIDMONGOL дотор `cert-svc` зориуд нийтлэгдээгүй. v2 stack нь national_id lookup-ыг auth callback-д шилжүүлсэн (privacy harvest reduce).
- ACL нь "per-operation" — нэг subsystem-ийн нэг сервисийн нэг үйлдэлд access өгөх боломжтой. Бүлгийн ACL ("security-server-owners") мөн дэмжигдэнэ, mgmt service-д ашигладаг.

**─────────────────────────────────────────────────**

---

## 4. Процесс / runtime архитектур

### 4.1 SS-ийн runtime процессүүд

```plantuml
@startuml ss-processes
title Security Server — runtime process layout (rp.gerege.mn жишээ)
skinparam component {
  BackgroundColor #F1F8E9
}

package "Ubuntu 24.04 host" {
  component "xroad-proxy\n(Jetty embedded HTTPS)" as proxy
  component "xroad-proxy-ui-api\n(Spring Boot, :4000)" as uiapi
  component "xroad-signer\n(Akka actors, SoftHSM2 client)" as signer
  component "xroad-confclient\n(60s timer)" as cc
  component "xroad-monitor\n(JMX exporters)" as mon
  component "xroad-addon-messagelog" as mlog
  database "PostgreSQL 16\nserverconf, messagelog" as pg
  folder "/etc/xroad/" as conf {
    artifact "configuration-anchor.xml"
    artifact "globalconf/MN/"
    artifact "keyconf.xml"
    artifact "signer/.softtoken.p12"
  }
}

proxy --> signer : "sign/verify\n(over IPC :5566)"
proxy --> mlog : "log msg + tspn"
proxy --> pg : "serverconf reads"
signer --> conf : "load keyconf"
cc --> conf : "write fresh globalconf"
uiapi --> proxy : "control + status"
uiapi --> pg : "config CRUD"
mon --> proxy : "JMX :2552"
mlog --> pg : "messagelog DB"

@enduml
```

### 4.2 CS-ийн runtime процессүүд

```plantuml
@startuml cs-processes
title Central Server — runtime process layout (cs.xroad.mn)
skinparam component {
  BackgroundColor #FFF3E0
}

package "Ubuntu 24.04 host" {
  component "xroad-center\n(Spring Boot, UI + APIs, :4000)" as center
  component "xroad-center-management-service\n(SOAP backend, :8085)" as mgmtsvc
  component "xroad-center-registration-service\n(SOAP backend, :8084)" as regsvc
  component "xroad-signer\n(softHSM)" as signer
  component "xroad-confclient" as cc
  component "xroad-nginx\n(:4001, :4002, :443, :80)" as ng
  database "PostgreSQL 16\ncenterui, messagelog" as pg
  folder "/etc/xroad/" as conf {
    artifact "globalconf/MN/shared-params.xml"
    artifact "globalconf/MN/private-params.xml"
    artifact "configuration-parts/*.ini"
    artifact "signer/.softtoken.p12"
  }
}

center --> pg : "CRUD members/SS/services"
center --> signer : "sign globalconf"
signer --> conf : "read/write keyconf"
mgmtsvc --> pg : "management_requests"
regsvc --> pg : "registration_requests"
ng --> regsvc : ":4001 proxy"
ng --> mgmtsvc : ":4002 proxy"
cc --> conf : "fetch own globalconf (self-loop)"

@enduml
```

### 4.3 Time-based процессүүд

```mermaid
gantt
    dateFormat HH:mm:ss
    axisFormat %M:%S
    title Recurring SS-side timers (one minute window)

    section confclient
    Refresh globalconf      :a, 00:00:00, 1s
    Idle                    :crit, 00:00:01, 59s

    section signer
    OCSP response cache check (per cert)  :b, 00:00:00, 1s
    OCSP query if stale     :c, 00:00:30, 2s
    Idle                    :crit, 00:00:32, 28s

    section cron
    cert-check.sh (timeserver)  :d, 00:00:00, 5s
    LE renewal check (nginx)    :e, 00:00:15, 5s
```

### 4.4 Мессеж дамжуулга — runtime sequence (low-level)

```mermaid
sequenceDiagram
    autonumber
    participant ISClient as IS client
    participant SSProxy as SS xroad-proxy
    participant SSSigner as SS xroad-signer (IPC :5566)
    participant Peer as Peer SS :5500
    participant TSA as TSA :443
    participant OCSP as ocsp.gerege.mn

    ISClient->>SSProxy: HTTP/REST<br/>X-Road-Client header
    SSProxy->>SSProxy: parse + build SOAP envelope
    SSProxy->>SSSigner: sign(envelope) IPC call
    SSSigner->>SSSigner: load SIGN priv from keyconf<br/>(softHSM token)
    SSSigner->>OCSP: cached OCSP good? else refresh
    OCSP-->>SSSigner: signed response (cached or fresh)
    SSSigner-->>SSProxy: signed envelope + cert
    SSProxy->>TSA: TSP request (hash of envelope)
    TSA-->>SSProxy: TimeStampToken
    SSProxy->>SSProxy: attach token to envelope
    SSProxy->>Peer: mTLS POST<br/>auth cert presented
    Peer-->>SSProxy: signed response (with own token)
    SSProxy->>SSProxy: verify response signature + TSP
    SSProxy-->>ISClient: REST body returned
```

---

## 5. Deployment архитектур

### 5.1 Хостуудын overview (NIIS Архитектур-стайл)

```d2
direction: down
title: Mongolia X-Road — physical deployment

cs_box: {
  label: "cs.xroad.mn\n38.180.203.234\nUbuntu 24.04"
  shape: rectangle
  cs_xroad: "xroad-center, mgmt-svc, reg-svc, nginx, postgres"
}

mgmt_box: {
  label: "mgmt.xroad.mn\n38.180.255.177"
  shape: rectangle
  mgmt_xroad: "xroad-proxy, xroad-proxy-ui-api, confclient, signer, postgres"
}

rp_box: {
  label: "rp.gerege.mn\n38.180.251.163"
  shape: rectangle
  rp_xroad: "xroad-proxy, ui-api, confclient, signer, monitor, postgres"
}

ssg_box: {
  label: "ss.gerege.mn\n66.181.175.134 (NAT 10.0.0.27)"
  shape: rectangle
  ssg_xroad: "xroad-proxy, ui-api, opmonitor, confclient, signer, postgres"
}

pay_box: {
  label: "ss.paygrid.mn\n38.180.254.231"
  shape: rectangle
  pay_xroad: "xroad-proxy, ui-api, confclient, signer, postgres"
}

ca_box: {
  label: "ca.gerege.mn\n38.180.136.97"
  shape: rectangle
  ca_stack: "nginx, gerege-ocsp (docker), gerege-crl, gerege-sign, eid-gerege-backend (Go), postgres"
}

ts_box: {
  label: "timeserver.mn\n38.180.203.29"
  shape: rectangle
  ts_stack: "Sigstore timestamp-server, nginx"
}

monitor_box: {
  label: "monitor.x-road.mn (== x-road.mn)\n38.180.242.76"
  shape: rectangle
  mon_stack: "Prometheus, node-exporter targets, Grafana"
}

cs_box -> mgmt_box: globalconf
cs_box -> rp_box: globalconf
cs_box -> ssg_box: globalconf
cs_box -> pay_box: globalconf
mgmt_box -> cs_box: management proxy

rp_box -> ca_box: HTTPS IS
ssg_box -> rp_box: SS-SS
pay_box -> rp_box: SS-SS

ca_box -> rp_box: "OCSP / CRL"
ca_box -> ssg_box: "OCSP / CRL"
ca_box -> mgmt_box: "OCSP / CRL"
ca_box -> pay_box: "OCSP / CRL"
ts_box -> rp_box: "timestamps"
ts_box -> ssg_box: "timestamps"
ts_box -> mgmt_box: "timestamps"
ts_box -> pay_box: "timestamps"

monitor_box -> cs_box: ":9100 scrape"
monitor_box -> mgmt_box: ":9100 scrape"
monitor_box -> rp_box: ":9100 scrape"
monitor_box -> ssg_box: ":9100 scrape"
monitor_box -> ca_box: ":9100 scrape"
monitor_box -> ts_box: ":9100 scrape"
```

### 5.2 X-Road багц + хувилбар

| Хост | xroad packages |
|---|---|
| cs.xroad.mn | xroad-centralserver, xroad-database-local, xroad-nginx, xroad-confclient, xroad-signer, xroad-center-management-service, xroad-center-registration-service |
| mgmt.xroad.mn | xroad-securityserver |
| rp.gerege.mn | xroad-securityserver |
| ss.gerege.mn | xroad-securityserver, xroad-securityserver-ee, xroad-opmonitor, xroad-addon-opmonitoring |
| ss.paygrid.mn | xroad-securityserver |
| ca.gerege.mn | (no X-Road; runs CA stack via Docker + Go backend) |
| timeserver.mn | (no X-Road; runs Sigstore TSA via systemd) |

X-Road хувилбар: **7.8.0** (NIIS upstream, Ubuntu 24.04).

### 5.3 Storage хэрэгцээ

| Хост | DB-ийн өсөлт/сар (өргөн зурвас) | Backup |
|---|---|---|
| cs.xroad.mn | <100 MB (centerui CRUD-only) | daily GPG → `/var/lib/xroad/backup/` |
| Member SS (any) | ~500 MB-2 GB (messagelog dominant) | daily GPG → `/var/lib/xroad/backup/` |
| ca.gerege.mn | ~200 MB (certificates + audit + sessions) | daily encrypted tarball off-host |
| timeserver.mn | <50 MB (no DB, just logs) | weekly tarball |

---

## 6. Өгөгдлийн архитектур

### 6.1 Гол өгөгдлийн биет (ER diagram)

```mermaid
erDiagram
    INSTANCE ||--o{ MEMBER : registers
    MEMBER ||--o{ SUBSYSTEM : owns
    MEMBER ||--o{ SECURITY_SERVER : owns
    SECURITY_SERVER ||--o{ SUBSYSTEM_BINDING : hosts
    SUBSYSTEM ||--o{ SUBSYSTEM_BINDING : bound
    SUBSYSTEM ||--o{ SERVICE : publishes
    SERVICE ||--o{ SERVICE_CLIENT : "ACL grants"
    SECURITY_SERVER ||--o{ AUTH_CERT : presents
    SECURITY_SERVER ||--o{ SIGN_CERT : signs
    SECURITY_SERVER ||--o{ TSP_ENTRY : uses
    INSTANCE ||--o{ APPROVED_CA : trusts
    INSTANCE ||--o{ APPROVED_TSA : trusts
    APPROVED_CA ||--o{ AUTH_CERT : issues
    APPROVED_CA ||--o{ SIGN_CERT : issues
    APPROVED_TSA ||--o{ TSP_RESPONSE : signs

    INSTANCE {
        string instanceId PK "MN"
    }
    MEMBER {
        string memberId PK "MN/COM/6235972"
        string name
        enum memberClass "COM | GOV | NEE"
    }
    SUBSYSTEM {
        string code PK
        enum status "SAVED | REGISTERED"
    }
    SECURITY_SERVER {
        string serverCode PK
        string ownerMemberId FK
        string publicIP
        string authCertHash
    }
    SUBSYSTEM_BINDING {
        string subsystemId FK
        string serverCode FK
        timestamp boundAt
    }
    SERVICE {
        string serviceCode PK
        string url
        string openApiUrl
    }
    SERVICE_CLIENT {
        string subjectId FK
        string serviceCode FK
        timestamp grantedAt
    }
    AUTH_CERT {
        string fingerprint PK
        string serverCode FK
        enum status "registered | active | deleted"
        timestamp validFrom
        timestamp validTo
    }
    SIGN_CERT {
        string fingerprint PK
        string memberId FK
        enum status
    }
    TSP_ENTRY {
        string serverCode FK
        string url
        string leafCertHash
    }
    APPROVED_CA {
        string caCertHash PK
        string ocspUrl
        string crlUrl
    }
    APPROVED_TSA {
        string leafCertHash PK
        string url
    }
    TSP_RESPONSE {
        bytes token
        string tsaCertHash FK
        timestamp signedAt
    }
```

### 6.2 globalconf XML файлууд

`/etc/xroad/globalconf/MN/`:

- `shared-params.xml` — гишүүд, security servers, approved CA, approved TSA, central services
- `private-params.xml` — managementService URL, auth-cert-reg endpoint, CS signing key info

CS UI бүх UI өөрчлөлтөд автоматаар бүгдийг **гарын үсэг тавьж re-generate хийнэ**. Дискт гар аргаар засаж болохгүй — confclient signature verify-д уначихна.

### 6.3 keyconf.xml (SS-side)

```xml
<keyConf>
  <device id="0">
    <token id="softToken-0">
      <key id="auth-key-1">
        <cert active="true" status="registered">
          <data>...base64 DER...</data>
        </cert>
      </key>
      <key id="sign-key-1">
        <cert active="true" status="registered" memberId="MN/COM/6235972">
          <data>...</data>
        </cert>
      </key>
    </token>
  </device>
</keyConf>
```

★ **Insight ─────────────────────────────────────**
- `active="false"` болсон cert байгаа боловч UI-д харагдаагүй тохиолдол гардаг. `rp.gerege.mn/HISTORY.md` 2026-04-19 болон `ss.gerege.mn/HISTORY.md` 2026-04-19 хоёулаа энэ алдаатай тулсан.
- Cert "registered" гэвэл CS-д аль хэдийн approval хийгдсэн гэсэн утга. "active" нь signer-д ашиглагдахад зориулагдсан.

**─────────────────────────────────────────────────**

---

## 7. Аюулгүй байдлын архитектур

### 7.1 STRIDE threat model

```mermaid
flowchart TB
    %% STRIDE — threats and mitigations

    subgraph threats[STRIDE threats]
        S["Spoofing<br/>(fake SS impersonates real SS)"]
        T["Tampering<br/>(modify msg in flight)"]
        R["Repudiation<br/>(member denies sending)"]
        I["Information Disclosure<br/>(eavesdrop SS-SS)"]
        D["DoS<br/>(SS-SS flood)"]
        E["Elevation of Privilege<br/>(consumer becomes admin)"]
    end

    subgraph mitigations[Architectural mitigations]
        M1["AUTH cert + mTLS<br/>+ globalconf attestation"]
        M2["SIGN cert over body+headers<br/>+ TSP timestamp"]
        M3["messagelog with cert+token<br/>(non-repudiation evidence)"]
        M4["TLS 1.2+, AES-GCM<br/>+ no public :5500 admin endpoint"]
        M5["UFW per-source-IP rules<br/>+ rate limit on :5500"]
        M6["CS UI is SSH-tunnel only<br/>+ form-login + future MFA"]
    end

    S --> M1
    T --> M2
    R --> M3
    I --> M4
    D --> M5
    E --> M6

    classDef threat fill:#FFEBEE
    classDef mitig fill:#E8F5E9
    class S,T,R,I,D,E threat
    class M1,M2,M3,M4,M5,M6 mitig
```

### 7.2 Сертификат болон түлхүүрийн lifecycle

```mermaid
stateDiagram-v2
    [*] --> KeyGenerated: SS generates AUTH/SIGN keypair
    KeyGenerated --> CSRSubmitted: SS prepares CSR
    CSRSubmitted --> Signed: gerege.mn signs<br/>(xroad_auth / xroad_sign profile)
    Signed --> Imported: SS imports .cer
    Imported --> Activated: operator clicks Activate
    Activated --> Registered: registered on CS<br/>(via mgmt-svc)

    Registered --> Active: in keyconf.xml<br/>active=true
    Active --> Stale: OCSP age > 3600s
    Stale --> Active: signer OCSP refresh

    Active --> Expiring: <30 days to expiry
    Expiring --> Active: renewal (new CSR, signed, imported)

    Active --> Revoked: OCSP revoke (compromise)
    Revoked --> [*]

    Active --> Expired: validity end
    Expired --> [*]
```

### 7.3 OCSP-ийн crash-проф архитектур

`gerege-ocsp` нь 2026-04-19-ний refactor (`gerege-mn-eid` commit `33f04ab`)-ийн дараа cache-гүй болсон. Хариу хариу dynamic sign хийнэ. Энэ нь "OCSP response is too old" алдааг бүрэн арилгасан.

```mermaid
sequenceDiagram
    autonumber
    participant Client as xroad-signer
    participant Nginx as nginx ocsp.gerege.mn
    participant Responder as gerege-ocsp
    participant View as v_all_certificates
    participant DB as postgres

    Client->>Nginx: POST /ocsp (DER)
    Nginx->>Nginx: root-POST rewrite → /ocsp
    Nginx->>Responder: proxy
    Responder->>View: SELECT WHERE issuer + serial
    View->>DB: UNION public.certificates,<br/>xroad.infra_certificates
    DB-->>View: row
    View-->>Responder: status, validity
    Responder->>Responder: build OCSPResponse<br/>thisUpdate=NOW
    Responder->>Responder: ECDSA P-256 sign<br/>(sub-ms)
    Responder-->>Nginx: signed DER
    Nginx-->>Client: 200 OK

    Note over Responder: cache removed 2026-04-19<br/>(commit 33f04ab)
```

### 7.4 X-Road infra schema (production isolation)

```plantuml
@startuml schema-iso
title PostgreSQL schema isolation on ca.gerege.mn (post 2026-04-20)

skinparam database {
  BackgroundColor #FFF8E1
}

package "public schema" {
  database "users" as users
  database "certificates\n(user AUTH/SIGN certs)" as user_certs
  database "audit_logs" as audit
  database "sessions" as sessions
  database "relying_parties\n(1 row: x-road gateway)" as rp_table
}

package "xroad schema" {
  database "infra_certificates\n(SS auth/sign certs, isolated)" as infra
  database "infra_certificates_audit\n(trigger-driven)" as infra_audit
}

view "v_all_certificates\n(UNION view for OCSP)" as union_view

union_view --> user_certs
union_view --> infra

note right of infra
  Protected from dev scripts:
  TRUNCATE public.certificates
  no longer touches X-Road certs.
end note

@enduml
```

---

## 8. Сүлжээний архитектур

### 8.1 Public IPv4 хаяг ба DNS

```mermaid
graph LR
    %% DNS → IP mapping

    DNS_CS["cs.xroad.mn"] --> IP1[38.180.203.234]
    DNS_CS2["cs.gerege.mn (legacy)"] --> IP1
    DNS_MGMT["mgmt.xroad.mn"] --> IP2[38.180.255.177]
    DNS_MGMT2["mgmt.gerege.mn (legacy)"] --> IP2
    DNS_RP["rp.gerege.mn"] --> IP3[38.180.251.163]
    DNS_SS["ss.gerege.mn"] --> IP4[66.181.175.134]
    DNS_PAY["ss.paygrid.mn"] --> IP5[38.180.254.231]
    DNS_CA["ca.gerege.mn"] --> IP6[38.180.136.97]
    DNS_CA2["gerege.mn"] --> IP6
    DNS_OCSP["ocsp.gerege.mn"] --> IP6
    DNS_CRL["crl.gerege.mn"] --> IP6
    DNS_SIGN["sign.gerege.mn"] --> IP6
    DNS_TS["timeserver.mn"] --> IP7[38.180.203.29]
    DNS_TS2["tsa.timeserver.mn"] --> IP7
    DNS_XR["x-road.mn"] --> IP8[38.180.242.76]
    DNS_TEST["test.gerege.mn"] --> IP8
    DNS_MON["monitor.x-road.mn"] --> IP8
```

### 8.2 NAT нөхцөл (ss.gerege.mn)

```d2
direction: right
title: ss.gerege.mn — NAT topology

internet: {
  shape: cloud
  label: "Public Internet"
}

router: {
  label: "ISP router\n66.181.175.134"
  shape: rectangle
  forwards: {
    label: "port-forward rules:\n22 → 10.0.0.27:22\n5500 → 10.0.0.27:5500\n5577 → 10.0.0.27:5577\n80 → 10.0.0.27:80\n443 → 10.0.0.27:443"
    shape: rectangle
  }
}

ss: {
  label: "ss.gerege.mn host\n10.0.0.27 / ens160\n(internal LAN)"
  shape: rectangle
  ufw: "UFW rules per port"
  proxy: "xroad-proxy listens *:5500, *:5577"
}

internet -> router
router -> ss: NAT forward

note: |md
  ⚠ Port-forward на routerа байх ёстой —
  UFW зөв ч router-д rule байхгүй бол гадаа ачаалал ороход timeout.
|
```

### 8.3 UFW анхдагч policy

```mermaid
flowchart LR
    %% UFW default deny + explicit allow pattern

    incoming[Incoming packet] --> def["Default: DROP"]
    def --> ssh{ssh from admin?}
    ssh -->|yes| allow_ssh[ALLOW :22]
    ssh -->|no| def2{":5500 from approved peer?"}
    def2 -->|yes| allow_5500[ALLOW :5500]
    def2 -->|no| def3{":4001/4002 per CS UFW?"}
    def3 -->|yes| allow_45xx[ALLOW]
    def3 -->|no| drop[DROP — logged]

    classDef allow fill:#E8F5E9
    classDef deny fill:#FFEBEE
    class allow_ssh,allow_5500,allow_45xx allow
    class drop deny
```

---

## 9. Интеграцийн загвар

### 9.1 IS ↔ SS integration patterns

**Producer SS (rp.gerege.mn) ↔ IS (`ca.gerege.mn /xroad/v1/*`):**

```mermaid
sequenceDiagram
    autonumber
    participant RP as rp.gerege.mn (producer SS)
    participant NG as ca.gerege.mn nginx
    participant BE as eid-gerege-backend (Fiber)

    RP->>NG: HTTPS /xroad/v1/auth/initiate<br/>X-Road-Client header
    NG->>NG: $remote_addr == 38.180.251.163?
    alt match
        NG->>NG: inject X-Gerege-SS-Token
        NG->>BE: proxy_pass
        BE->>BE: middleware verifies token
        BE->>BE: persist X-Road-Client in audit log
        BE-->>NG: business response
        NG-->>RP: 200 OK
    else other IP
        NG--xRP: 403 X-Road IS endpoint restricted
    end
```

**Consumer SS (ss.paygrid.mn) ↔ IS (paygrid.mn 10.0.0.27):**

mTLS, нь mutual TLS. SS-аас IS-руу хандаж IS-ийн TLS cert-ийг pin хийдэг (uploaded under "Information System TLS certificates").

```mermaid
sequenceDiagram
    autonumber
    participant SS as ss.paygrid.mn
    participant IS as paygrid.mn (38.180.254.229) :8443

    SS->>IS: TLS Client Hello
    IS-->>SS: Server Hello + IS cert
    SS->>SS: verify IS cert hash<br/>against uploaded IS TLS cert
    SS->>IS: TLS Client Cert (ECDSA P-256, generated on paygrid.mn)
    IS->>IS: verify SS client cert
    SS->>IS: HTTPS POST /xroad-call
    IS-->>SS: 200 OK
```

### 9.2 Хадгалагдаагүй intentional gaps

- **Op-Monitor UI dashboard** — sensor аль хэдийн идэвхтэй (`xroad-opmonitor`-ийг ss.gerege дээр суулгасан), гэхдээ UI байхгүй. Phase-3.
- **Federation** — TRUST POLICY сегмент бэлэн биш. 2027 шинэ зорилго.
- **Op-Monitor central scraping** — Prometheus `xroad-nodes` job нь node_exporter-ыг scrape хийдэг, гэхдээ X-Road-ийн өөрийн opmonitor JMX-уудыг scrap хийдэггүй. Phase-3.

---

## 10. Архитектурын шийдвэрийн бүртгэл (ADR)

Бид архитектурын шийдвэр бүрийн **why**-г бүртгэдэг — кодын комментоос илүү, гадаргуугаас илүү. Хамгийн чухлууд:

### ADR-001: Why Single Instance, Not Federation (2026-04)
- **Status**: Accepted
- **Context**: NIIS-ийн federation нь гадаад instance-уудыг trust policy-аар холбохыг зөвшөөрнө. Бид анх Estonia-тай холбохыг бодсон.
- **Decision**: Phase 1-д single instance. Federation бол 2027+ дараах stage.
- **Consequence**: Mongolia-ийн citizens-ийн identity нь зөвхөн `MN` instance-ийн дотор хүчинтэй. Cross-border eID нь өөр channel.

### ADR-002: Why ca.gerege.mn Hosts Everything-PKI (2026-04)
- **Status**: Accepted
- **Context**: PKI (Root CA + Issuing CA + OCSP + CRL + sign portal) + IS endpoint бүгдийг нэг хост дээр байрлуулах нь "single point" гэх concern үүсгэдэг.
- **Decision**: Эхний фазад бүгдийг ca.gerege.mn-д. Шалтгаан: operational complexity, single team, secret custody. Phase-3-д Root CA-г offline зөөвөрт шилжүүлнэ.
- **Consequence**: ca.gerege.mn-ий боловсон availability бүх SS-ийн "OCSP fresh" require-д шууд нөлөөлдөг. Mitigation: OCSP responder dynamic sign (cache-гүй), backup daily, hot-restart-able.

### ADR-003: Why Separate TSA Issuing CA (2026-04-19)
- **Status**: Accepted
- **Context**: Sigstore TSA нь EVERY non-root cert-д `id-kp-timeStamping` EKU байх require хийдэг. Gerege Issuing CA нь general purpose, EKU-гүй.
- **Decision**: Тусдаа `Gerege TSA Issuing CA` intermediate үүсгэсэн. Зөвхөн TSA leaf-д ашиглана.
- **Consequence**: 2 intermediate бүтэцтэй PKI. Operational cost маш бага, шаардлагын зөв.

### ADR-004: Why X-Road Gateway Single-Row Refactor (2026-04-19)
- **Status**: Accepted
- **Context**: Анх `eid-gerege-backend` нь Service-clients ACL-тай parallel `xroad_subsystems` table-той байсан. Хоёр газар sync хийх error-prone.
- **Decision**: `xroad_subsystems` table-г DROP хийсэн. `relying_parties` дотор зөвхөн 1 row (`00000001-0000-4000-8000-000000000000` "X-Road Gateway"). `X-Road-Client` header нь sessions.xroad_client + audit_logs.xroad_client-д л хадгалагдана.
- **Consequence**: Service-clients ACL = single source of truth. Partner onboarding нь "rp UI-д access нэмэх" гэж 1 алхам болсон. Trust radius нь nginx IP-pin + X-Gerege-SS-Token-аар хязгаарлагдсан.

### ADR-005: Why xroad.infra_certificates Schema (2026-04-20)
- **Status**: Accepted
- **Context**: Dev environment-ийн SQL script нь production `public.certificates` rows-уудыг bulk revoke хийсэн, X-Road infra cert-уудыг хальт цохисон. SS-SS handshake тэр дороо унасан.
- **Decision**: X-Road infra cert-уудыг тусдаа `xroad.infra_certificates` schema-д шилжүүлсэн (migration `015_xroad_infra_schema.sql`). `v_all_certificates` UNION view-аар OCSP responder ижил interface-тай хэвээр үлдсэн.
- **Consequence**: `TRUNCATE public.certificates` нь одоо infra cert-уудыг хальдахгүй. Forensic: trigger-driven `xroad.infra_certificates_audit` бүх UPDATE/INSERT/DELETE-ийг session_user + inet_client_addr + application_name-ийн хамт лог-д.

### ADR-006: Why Mermaid for All Docs (2026-05-14)
- **Status**: Accepted
- **Context**: Анх ASCII диаграмм `git diff`-д уншиж амар гэж policy байсан. x-road.mn/ нэгэнт Mermaid-аар pre-rendered SVG-ийг public-д үзүүлэхэд ашигладаг.
- **Decision**: Бүх doc-д Mermaid (Plus PlantUML/D2 өөрт нь тохирох тохиолдолд) standard болгосон. ASCII бол code listing-д л.
- **Consequence**: GitHub шууд render, public site дамжуулдаг, slide-д ашиглаж болно.

### ADR-007: Why Port 4000 Pre-Prod Showcase Was Reverted (2026-04-22)
- **Status**: Accepted
- **Context**: 2026-04-20-нд showcase зориулж бүх 4 хост (cs/mgmt/rp/ss) дээр UFW port 4000-ийг public нээсэн. 2 хоногийн дараа буцаасан.
- **Decision**: Port 4000 нь хэзээ ч public байх ёсгүй. SSH tunnel л баталгаажсан хандалт. 2026-05-14-нд яамны танилцуулгад зориулж дахин нээж, мөн өдөр буцаах товлосон.
- **Consequence**: `ss.paygrid.mn/README.md`-д "Do NOT open port 4000 to public — even briefly. The 2026-04-20 pre-prod showcase exposure ... is not repeated here" гэж дам бичигдсэн.

---

## 11. Гадаад хамаарал ба upstream

### 11.1 NIIS upstream

X-Road кодын тэргүүн нь NIIS (Nordic Institute for Interoperability Solutions). Бид:
- Ubuntu 24.04 LTS-д зориулсан Debian packages-ийг шууд ашигладаг (no custom build).
- Хувилбар 7.8.0-1 нь stable-track.
- Шинэ хувилбар (7.9+) гарахад staging-д тест хийгээд production-д рул-аут (Phase-3).

### 11.2 Sigstore upstream

TSA нь Sigstore community-ийн `timestamp-authority` бинарийг v2.0.6-д pin хийсэн. Шинэ хувилбар certchain validation rule-уудыг өөрчилдөг тул pin шинэчлэхийн өмнө sandbox-д тест хэрэгтэй.

### 11.3 Let's Encrypt

`ca.gerege.mn`, `tsa.timeserver.mn`, `x-road.mn`, `monitor.x-road.mn` бүгд LE-ээс TLS cert авдаг. ISRG Root X1/X2 нь mobile-side TLS pin-ийн target. LE-ийн уламжлалт rotation policy: жилд нэг intermediate (R10/R11) болж нэг тарихаа байгаа.

### 11.4 Docker images

- `gerege-ocsp`, `gerege-crl`, `gerege-sign`, `eid-gerege-backend` бүгд `gerege-mn-eid` репозитороор build хийгдэж private registry-аас pull хийгдэнэ.
- Container restart нь in-memory key-уудыг дахин load хийдэг — operational consideration.

### 11.5 PostgreSQL upstream

PG 16 LTS суурьтай. Бүх xroad-related DB-уудыг тус тусын кластер дотор. Замбараагүй upgrade-аас зайлсхийхийн тулд auto-update идэвхгүй (`/etc/apt/apt.conf.d/50unattended-upgrades`).

---

*Энэ архитектур гарын авлагыг 2026-05-15-нд бичсэн. Дараагийн засвар: гишүүн SS нэмэгдэх, federation policy шинэчлэх, эсвэл хост шилжүүлэх үед бичсэн ажилтан энэ файлыг шинэчилнэ үү.*
