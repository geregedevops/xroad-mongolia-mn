# ИГ-CS — Central Server суулгах гарын авлага (Installation Guide)

> **Doc code:** IG-CS — analogous to NIIS `IG-CS`.
> **Scope:** Шинээр Mongolia X-Road instance (эсвэл шинэ environment-д) Central Server-ийг суулгах процесс. Production-grade install. Backup, monitoring, hardening орсон.
> **Audience:** Системийн админ, SRE баг. NIIS upstream packages-тэй танил байх ёстой.

---

## Агуулга

1. [Пререкизит](#1-пререкизит)
2. [Суулгах процессын тойм](#2-суулгах-процессын-тойм)
3. [OS бэлдэх](#3-os-бэлдэх)
4. [Сүлжээ + UFW](#4-сүлжээ--ufw)
5. [X-Road packages суулгах](#5-x-road-packages-суулгах)
6. [Эхний тохиргооны wizard](#6-эхний-тохиргооны-wizard)
7. [Trust services тохируулах](#7-trust-services-тохируулах)
8. [Backup, monitoring](#8-backup-monitoring)
9. [Hardening checklist](#9-hardening-checklist)
10. [Verification + sign-off](#10-verification--sign-off)

---

## 1. Пререкизит

| Зүйл | Шаардлага | Тэмдэглэл |
|---|---|---|
| OS | Ubuntu 24.04 LTS (Server) | Production-grade. 22.04 ажиллах боловч NIIS-аас 24.04-д шилжсэн. |
| CPU | ≥4 vCPU | Globalconf sign + UI traffic. |
| RAM | ≥8 GB | Postgres + Java + nginx. |
| Disk | ≥80 GB | OS 20G + Postgres + logs + backups. |
| Static public IP | mandatory | гишүүд `cs.xroad.mn:4001`-аар хандана. |
| DNS A record | `cs.xroad.mn` → IP | TTL ≤300s эхэн үеийн өөрчлөлтөд. |
| Time sync (NTP) | `chronyd` идэвхтэй | RFC 3161 timestamping + cert validity-д чухал. |
| Хууль зүйн эрх мэдэл | CS instance authority (ҮДТ) | `mgmt.xroad.mn` нь яаманд харьяалагдсан. |

## 2. Суулгах процессын тойм

```mermaid
flowchart TB
    Start([Start]) --> OS[OS install + harden]
    OS --> NET[Network + UFW]
    NET --> PKG[apt install xroad-centralserver]
    PKG --> WIZ[Initial Configuration Wizard]
    WIZ --> TRUST[Add approved CA + TSA]
    TRUST --> MGMT[Configure management-service]
    MGMT --> BACK[Setup GPG backup]
    BACK --> MON[Add to monitoring]
    MON --> VER[Verify all signals]
    VER --> Done([Sign-off + handover])

    classDef phase fill:#E3F2FD
    class OS,NET,PKG,WIZ,TRUST,MGMT,BACK,MON,VER phase
```

## 3. OS бэлдэх

```bash
# 1) latest patches
sudo apt update && sudo apt full-upgrade -y

# 2) timezone
sudo timedatectl set-timezone Asia/Ulaanbaatar

# 3) NTP via chrony
sudo apt install -y chrony
sudo systemctl enable --now chrony
chronyc tracking   # verify offset < 100ms

# 4) sudo + admin user
sudo useradd -m -G sudo -s /bin/bash opsadmin
sudo passwd opsadmin
sudo mkdir -p ~opsadmin/.ssh
# put admin pubkey:
echo "ssh-ed25519 AAAA...  ops" | sudo tee -a ~opsadmin/.ssh/authorized_keys
sudo chown -R opsadmin:opsadmin ~opsadmin/.ssh
sudo chmod 700 ~opsadmin/.ssh; sudo chmod 600 ~opsadmin/.ssh/authorized_keys

# 5) sshd hardening
sudo sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

## 4. Сүлжээ + UFW

```bash
sudo apt install -y ufw
# default deny incoming, allow established + outgoing
sudo ufw default deny incoming
sudo ufw default allow outgoing

# admin SSH pinned to admin source IP(s)
sudo ufw allow from 38.180.242.76 to any port 22 proto tcp comment "ops admin SSH"

# Public HTTP/HTTPS for managementservices.wsdl + Let's Encrypt
sudo ufw allow 80/tcp  comment "ACME challenge"
sudo ufw allow 443/tcp comment "WSDL + LE-issued TLS"

# globalconf (4001) and mgmt-service (4002) — per-member rules added later
# DO NOT open 4000/tcp publicly — admin UI is SSH-tunnel only

sudo ufw enable
sudo ufw status numbered
```

```mermaid
graph LR
    %% UFW initial state on CS

    INET[Internet] -->|22 from admin IP only| SSH[allow]
    INET -->|80, 443 public| WEB[allow]
    INET -->|4001 per-member rule| FAIL_DEFAULT[default DROP]
    INET -->|4002 per-member rule| FAIL_DEFAULT
    INET -->|4000 anywhere| DROP_4000[DROP]
    INET -->|other| DROP[default DROP]

    classDef allow fill:#E8F5E9
    classDef deny fill:#FFEBEE
    class SSH,WEB allow
    class DROP_4000,DROP,FAIL_DEFAULT deny
```

## 5. X-Road packages суулгах

NIIS upstream repo нэмэх:

```bash
# add NIIS apt repo
curl -fsSL https://artifactory.niis.org/api/gpg/key/public | sudo gpg --dearmor -o /usr/share/keyrings/niis-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/niis-archive-keyring.gpg] https://artifactory.niis.org/xroad-release-deb noble-current main" | sudo tee /etc/apt/sources.list.d/xroad.list
sudo apt update

# Install CS bundle
sudo apt install -y xroad-centralserver
```

Package install нь дараахыг автоматаар хийнэ:
- Postgres 16 cluster (`/etc/postgresql/16/main/`)
- `xroad-center` Java systemd unit
- `xroad-center-management-service` + `xroad-center-registration-service`
- `xroad-confclient`, `xroad-signer`
- `/etc/xroad/conf.d/local.ini` skeleton
- nginx vhost конфиг

```mermaid
sequenceDiagram
    autonumber
    participant Apt as apt install
    participant Post as postinst hook
    participant PG as postgres init
    participant Sys as systemd

    Apt->>Apt: pull xroad-centralserver + deps
    Apt->>Post: run postinst
    Post->>PG: createdb centerui, messagelog
    Post->>PG: apply migrations
    Post->>Sys: enable xroad-center.service
    Post->>Sys: enable xroad-confclient.timer
    Post->>Sys: enable xroad-signer.service
    Sys-->>Apt: services up
    Apt-->>Apt: "Open https://localhost:4000 to finish setup"
```

## 6. Эхний тохиргооны wizard

```bash
# Open browser via tunnel
ssh -L 14000:localhost:4000 cs.xroad.mn
# in browser → https://localhost:14000
```

Wizard steps:

1. **Admin user** — `xrdadmin` (strong password; store in `reference_cs_secrets.md`).
2. **Instance identifier** — `MN`.
3. **Central server address** — `cs.xroad.mn` (public DNS).
4. **Software token PIN** — 8+ chars; store in secrets file.
5. **Database** — local, accept defaults.
6. **Initial configuration sign-off** — review summary, click Finish.

After wizard, CS automatically:
- Generates CS signing key in softHSM token.
- Creates `/etc/xroad/globalconf/MN/` with empty `shared-params.xml` skeleton.
- Starts confclient self-loop (CS distributes its own globalconf).

## 7. Trust services тохируулах

### 7.1 Approved CA нэмэх

```mermaid
sequenceDiagram
    autonumber
    actor Op as Operator
    participant UI as CS UI :4000
    participant DB as centerui DB
    participant FS as globalconf/MN/shared-params.xml
    participant SS as xroad-signer

    Op->>UI: Trust Services → Certification Services → Add
    Op->>UI: upload root-ca.pem + issuing-ca.pem
    Op->>UI: configure intermediates, OCSP URL, CRL URL
    UI->>DB: insert approved_ca rows
    UI->>SS: trigger re-sign globalconf
    SS->>FS: write new signed shared-params.xml
    Note over FS: all member SSes pick up<br/>via confclient timer (~60s)
```

Конкрет: 
- Root CA: `gerege.mn:/opt/.../pki/root-ca.pem`
- Issuing CA: `gerege.mn:/opt/.../pki/issuing-ca.pem`
- OCSP URL: `https://ocsp.gerege.mn/ocsp`
- CRL URL: `https://crl.gerege.mn/issuing-ca.crl`

### 7.2 Approved TSA нэмэх

UI → Trust Services → Timestamping Services → Add:
- Name: `TimeServer.mn`
- URL: `https://tsa.timeserver.mn/`
- Cert: `/opt/tsa-certs/leaf-cert.pem` (scp from timeserver.mn)

### 7.3 Management service тохиргоо

UI → Settings → System parameters → management-service:
- `managementServiceProviderId`: `MN/GOV/6806252/MANAGEMENT` (after mgmt SS registered)

`private-params.xml`-ийн `<managementService>` URL `https://cs.xroad.mn:4001/managementservice/` хадгалагдана (auth-cert-reg endpoint).

## 8. Backup, monitoring

### 8.1 GPG backup setup

```bash
# generate GPG key for backup encryption
sudo -u xroad gpg --homedir /etc/xroad/gpghome --batch --generate-key <<EOF
%no-protection
Key-Type: RSA
Key-Length: 4096
Name-Real: cs-xroad-backup
Expire-Date: 0
%commit
EOF

# get keyid
sudo -u xroad gpg --homedir /etc/xroad/gpghome --list-keys

# put keyid into local.ini
sudo vi /etc/xroad/conf.d/local.ini
# [center]
# backup-encryption-keyid = ABCDEF0123456789
```

Default daily backup → `/var/lib/xroad/backup/`. Keep encrypted off-host copy.

### 8.2 Monitoring add

`monitor.x-road.mn` (`38.180.242.76`) дээр Prometheus:

```bash
# on CS:
sudo apt install -y prometheus-node-exporter
sudo ufw allow from 38.180.242.76 to any port 9100 proto tcp comment "prometheus scrape"

# on monitor.x-road.mn:
# add to /opt/xroad-monitor/prometheus.yml under job: xroad-nodes
#   - targets: ['cs.xroad.mn:9100']
sudo systemctl reload prometheus
```

```mermaid
graph LR
    PROM[monitor.x-road.mn Prometheus] -->|"scrape :9100"| CS[cs.xroad.mn node_exporter]
    PROM --> Alert[Alertmanager]
    Alert --> Email[Operator email]
    Alert --> Slack[Slack #xroad-ops]
```

## 9. Hardening checklist

| Зүйл | Status |
|---|---|
| ✅ `:4000` НЭ public (зөвхөн SSH tunnel) | mandatory |
| ✅ Strong `xrdadmin` password (16+, mixed) | mandatory |
| ✅ SSH key-only (PasswordAuthentication no) | mandatory |
| ✅ `:22` UFW pinned to admin IP | mandatory |
| ✅ Auto-update enabled for security (unattended-upgrades) | recommended |
| ✅ GPG backup keyid set | mandatory |
| ✅ NTP `chrony` synced | mandatory |
| ✅ Daily encrypted backup off-host | mandatory |
| ☐ MFA для xrdadmin UI login (Phase-3) | future |
| ☐ Centralized log shipping | future |

## 10. Verification + sign-off

```bash
# 1) all services healthy
systemctl status xroad-center xroad-center-management-service xroad-center-registration-service xroad-confclient xroad-signer postgresql@16-main xroad-nginx
# all should be active (running)

# 2) ports listening as expected
sudo ss -tlnp | grep -E ':(4000|4001|4002|80|443) '
# expect *:4000, *:4001, *:4002, *:80, *:443

# 3) globalconf successfully signed
ls -la /etc/xroad/globalconf/MN/
cat /etc/xroad/globalconf/MN/shared-params.xml.metadata
# expect signature_algorithm + expiration_date present

# 4) UI accessible
ssh -L 14000:localhost:4000 cs.xroad.mn
# open https://localhost:14000 → login as xrdadmin

# 5) public endpoints respond
curl -sI https://cs.xroad.mn/managementservices.wsdl | head -1   # 200 OK
curl -sk -o /dev/null -w '%{http_code}\n' https://cs.xroad.mn:4001/   # 400 or 200 expected
```

```mermaid
graph TB
    %% Sign-off gating

    START([Install complete]) --> CHK1{All systemd units green?}
    CHK1 -->|no| FIX1[Fix unit, retry]
    CHK1 -->|yes| CHK2{Ports listening?}
    CHK2 -->|no| FIX2[Check listen IF / UFW]
    CHK2 -->|yes| CHK3{globalconf signed?}
    CHK3 -->|no| FIX3[Check signer + softHSM]
    CHK3 -->|yes| CHK4{UI reachable via tunnel?}
    CHK4 -->|no| FIX4[Check SSH + nginx]
    CHK4 -->|yes| CHK5{Public WSDL serves?}
    CHK5 -->|no| FIX5[Check LE cert + nginx vhost]
    CHK5 -->|yes| SIGN([Sign-off complete])

    FIX1 --> CHK1
    FIX2 --> CHK2
    FIX3 --> CHK3
    FIX4 --> CHK4
    FIX5 --> CHK5

    classDef ok fill:#E8F5E9
    classDef fix fill:#FFEBEE
    class CHK1,CHK2,CHK3,CHK4,CHK5,SIGN ok
    class FIX1,FIX2,FIX3,FIX4,FIX5 fix
```

---

*Sign-off doc-уудыг `cs.xroad.mn/HISTORY.md`-д "Install" хэсэгт оруулна. CS install бол **once-per-instance** үйл явдал — энэ playbook нь disaster recovery эсвэл шинэ environment бэлдэх үед л дахин ажилладаг.*
