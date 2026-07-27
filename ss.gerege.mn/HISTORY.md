# ss.gerege.mn — operational history

## 2026-04 — Pre-existing install (Gerege Core LLC) re-used a long-lived VM

Was the first member SS deployed for the Mongolia X-Road instance after CS came up. Owner client `MN/COM/6884857` (Gerege Core LLC) registered via the standard mgmt-service flow.

**Reused VM — not a fresh install.** `/var/log/apt/history.log.6.gz` shows apt activity from `2024-04-23` (Ubuntu live-installer bootstrapping), then sporadic `unattended-upgrades` and `apt upgrade` across 2024-2025, and finally the X-Road install on top. The root bash history (`apt update && apt upgrade`, `pvresize /dev/sda3`, `lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv`, `resize2fs`) shows the disk was grown to make room for X-Road before the package install. **Watch out:** if this SS is ever wiped and re-provisioned, don't assume the kernel + networking config is the stock Ubuntu 24.04 — there have been manual `/etc/netplan/50-cloud-init.yaml` edits and custom `sshd_config` on this host.

**Install automation.** This SS was installed using an `expect`-driven helper (`/tmp/xroad-install.exp` in `~grgdev/.bash_history`) that debconf-answers the `xroad-securityserver-$VARIANT` wizard prompts (admin username, DB URL, CN, SAN, JVM memory profile). That script is reusable for any future Ubuntu 24.04 member SS — the bash history entry is the only record of it today; the script itself was a scratch `/tmp/` file and is now gone.

## 2026-04-19 — Adding TEST-DEMO subsystem hit `Member 'SUBSYSTEM:MN/COM/6884857/TEST-DEMO' has no suitable certificates`

**Symptom:** Clicking Register on the new TEST-DEMO subsystem gave this error from `signer.internal_error`.

**Root cause:** Same OCSP staleness issue we saw on rp — the `gerege-ocsp` container had an old response (~1.5h) and the X-Road freshness window is 3600s. The signer dropped the SS's SIGN cert from its "suitable" set, so any new client request that needs to be signed fails.

Stack trace from `/var/log/xroad/signer.log`:
```
incorrect_validation_info: OCSP response is too old (thisUpdate: 2026-04-19T03:10:25Z)
  at OcspVerifier.verifyValidityAt(OcspVerifier.java:193)
  at OcspClientWorker.queryCertStatus(OcspClientWorker.java:278)
```

**Fix:**
```bash
ssh gerege.mn 'docker restart gerege-ocsp'
ssh ss.gerege.mn 'sudo systemctl restart xroad-signer'
```

After that the signer's OCSP refresh job logged "OCSP-response refresh cycle successfully completed" and the cert was suitable again. Retried Register → REGISTERED in seconds.

**Watch out:** This pattern (OCSP age > freshness → certs become "unsuitable") will hit ANY signing-cert use, not just registration. If you ever see "no suitable certificates", restart the OCSP container first, then the signer here.

## 2026-04-19 — `Security server has no valid authentication certificate` after the above fix

**Symptom:** Right after the OCSP/signer dance, retrying Register surfaced this new error.

**Root cause:** AUTH cert had `active="false"` in `keyconf.xml`. Same as on rp.gerege.mn — the Initial Config Wizard imports the cert as registered but does not auto-activate it.

**Fix:** UI → Keys and Certificates → expand AUTH key → click cert → Activate.

**Watch out:** Activate AUTH AND SIGN certs immediately after import. The UI gives no big red warning if either is inactive.

## 2026-04-19 — Consumer call to GEREGE-ID returned `Client (SUBSYSTEM:MN/COM/6884857/TEST-DEMO) specifies HTTPS but did not supply TLS certificate`

**Symptom:** From an internal IS (test.gerege.mn backend) calling `http://localhost/r1/MN/COM/6235972/GEREGE-ID/auth-svc/auth/initiate` with `X-Road-Client: MN/COM/6884857/TEST-DEMO`, got HTTP 500 with this message.

**Root cause:** TEST-DEMO subsystem's "Internal Servers → Connection type" was the default HTTPS (with auth). The IS (test.gerege.mn) was calling over plain HTTP from inside its docker network, so no client TLS cert was offered.

**Fix:** UI → Clients → TEST-DEMO → Internal Servers → Connection type → HTTP → Save.

**Watch out:** For a CONSUMER subsystem this knob controls how the local IS reaches the SS. For a PRODUCER subsystem the knob is irrelevant — provider-role connection type is inferred from the published service URL. Don't confuse the two.

## 2026-04-19 — UFW closed port 80 to test.gerege.mn host

**Symptom:** `curl http://ss.gerege.mn/r1/...` from `x-road.mn` (38.180.242.76) failed with timeout.

**Root cause:** UFW had explicit rules for `5500/tcp` (SS-SS message) and `5577/tcp` (OCSP) but not for the local consumer REST gateway on `80/tcp`.

**Fix:**
```bash
sudo ufw allow from 38.180.242.76 to any port 80 proto tcp comment "test.gerege.mn IS"
```

**Watch out:** This SS uses ports `80/tcp` and `443/tcp` for the consumer REST gateway (custom — X-Road defaults are 8080/8443). Any new IS host needs an explicit UFW rule. Don't open port 80 to public — anyone with network access can send X-Road requests as TEST-DEMO and consume our quota.

## 2026-04-20 — Pre-prod showcase opened port 4000/tcp to public Internet

Same change as on cs/mgmt/rp: `ufw allow 4000/tcp comment 'pre-prod showcase 2026-04-20'`. Consumer SS admin panel was reachable from the Internet until **2026-04-22**, when the rule was deleted in a single batch across all four MN hosts. Admin UI access is back to `ssh -L 14002:localhost:4000 ss.gerege.mn`.

## 2026-04-22 — State-verification snapshot (taken for monorepo audit)

- **X-Road package set matches the documented "ee + opmonitor" profile**: `xroad-securityserver`, `xroad-securityserver-ee` (Estonian country profile), `xroad-opmonitor`, `xroad-addon-opmonitoring`, plus the common security-server stack. Running services confirm `xroad-opmonitor` is the differentiator vs mgmt/rp.
- **Postgres has an extra `op-monitor` database** (6 dbs total: `messagelog`, `op-monitor`, `postgres`, `serverconf`, `template0`, `template1`). Same cluster, same credentials pattern.
- **Listening ports include `:80` and `:443` (non-standard)** — consumer REST gateway on standard HTTP/HTTPS ports, not the X-Road default `8080`/`8443`. Also `:5500` (SS peer message), `:5577` (OCSP peer), `:4000` (showcase), plus opmonitor on loopback ports `2080`/`2081`.
- **UFW rule-set** (as of snapshot): `22/tcp` Anywhere (not admin-pinned like cs — an inconsistency worth fixing), `5500/tcp`+`5577/tcp` Anywhere (SS peer), `8080/tcp`+`8443/tcp` from `10.0.0.0/24` (LAN consumers), `80/tcp` from `38.180.242.76` (test.gerege.mn IS only), `4000/tcp` Anywhere (showcase).
- **Internal IP**: `10.0.0.27/24` on `ens160` — this host is behind NAT; public `66.181.175.134` is the router's forwarding address, not on the SS itself.
- **Softtoken has 2 active p12s** (`2089…1CD3`, `DC1C…4A09`, both 2026-04-19 04:50). Matches one AUTH + one SIGN key round. `.softtoken.p12` 2026-04-19 04:43 is the seal key.

## 2026-07-08 — Admin UI (:4000) re-opened to public Internet (showcase reinstated)

**Change:** `ufw allow 4000/tcp` re-added on ss.gerege.mn — reinstating the 2026-04-20 exposure reverted 2026-04-22. Done at operator request, completing the set alongside cs.xroad.mn, mgmt.xroad.mn and rp.gerege.mn.

**NAT caveat (host-specific — this is the one that bit before):** ss.gerege.mn sits behind the ISP router `66.181.175.134`, which only port-forwards `22, 5500, 5577, 80, 443` to the host at `10.0.0.27`. `ufw allow 4000/tcp` on the host is necessary but NOT sufficient — without a matching router port-forward rule for `4000/tcp`, connections from the Internet time out (see the 2026-05-14 NAT-trap note in `README.md`). To actually expose the UI you must add the `4000 → 10.0.0.27:4000` forward on the router as well.

**Risk (unchanged):** Form-login only, no mTLS / IP allow-list / WAF. This is the GEREGE-WALLET consumer-side SS; a compromised UI session can alter its client list and service-access configuration.

**Watch out:** Re-tighten when the showcase closes — remove BOTH the router port-forward rule AND `sudo ufw delete allow 4000/tcp` (IPv4 + IPv6), verify with `ufw status`. Baseline access is `ssh -L 14002:localhost:4000 ss.gerege.mn`.

## 2026-07-27 — globalconf dead since 2026-07-05: cs.xroad.mn re-keyed its internal signing key

Found while doing a routine login check on this host. `xroad-confclient` had been failing every 60 s and the local globalconf was **22 days expired** (`shared-params.xml.metadata` → `expirationDate 2026-07-05T21:21:14Z`). Three distinct failures stacked up, visible as per-day error-code counts in `/var/log/xroad/configuration_client.log*`:

| From | errorCode | Cause |
|------|-----------|-------|
| 07-05 | `GLOBAL_CONF_GET_VERSION_FAILED` | cs now serves a **self-signed** web cert (`CN=cs.xroad.mn`, issuer=itself, 2026-07-22 → 2046). The anchor's `https://` download URL fails PKIX path building. Not fatal on its own — the `http://` URL in the same anchor returns 200 and the conf is signature-verified anyway. |
| 07-08 | `GLOBAL_CONF_OUTDATED` | local copy past its expire-date |
| 07-22 | `GLOBAL_CONF_MISSING_VERIFICATION_CERT` | **the real blocker** |

**Root cause (proved, not guessed).** cs.xroad.mn regenerated its internal configuration **signing** key around 2026-07-22. The anchor on this host (`generatedAt 2026-04-18T21:42:43.919Z`) carries two `internalSigningKey` certs; cs now signs with a third:

```
anchor  sha512(b64): 6SjJSfBKPE3AgUHZdVMBJqWuycbWTQpR86CFZsGhUml1YyGdId23/mIh3kkoi2WVu4cvNgWAZ6PuEcdZ7bZXNw==
                     jsTutDagfp0SyX/kqdEXpTKQ6ktBdore+kiAEX5rBySfCtVqVo0YLJ46Mm/wLdM8Jy7WgrT5Ocs6fhEzAEcT8A==
served  sha512(b64): SjBD2johaVYOMnE2xW7uxoJK4vzpKmAoMKNpLEPKW5CJ6ckD4lkU8MJ3sJ45TXSRYIY7KaOJ31DK2PF/HG+ZYw==
                     → no overlap
```

cs itself is healthy and generating fresh conf (`Expire-date` ~1 h ahead of each fetch). **Every member SS in the instance is affected** — `rp.gerege.mn`, `mgmt.xroad.mn`, `ss.paygrid.mn` all hold the same 2026-04-18 anchor, and all four anchors committed in this repo are that same dead one. The fix is a new anchor from cs on each host; there is no SS-side workaround, because the signed conf envelope carries only the signing cert's *hash*, never the cert.

**Watch out:** an expired globalconf means the SS refuses to route X-Road messages. Nobody noticed for 22 days because this SS had served no traffic since 2026-05-15 (`proxy.log`).

## 2026-07-27 — Wiped and rebuilt as a Gerege Systems LLC SS (`GEREGE-SS-1`) — registration still pending

Done at operator request: destroy the Gerege Core LLC install and rebuild this host as a security server for **Gerege Systems LLC `MN/COM/6235972`**, server code **`GEREGE-SS-1`**, keeping the previous SS's config profile and a publicly reachable admin UI.

**Live state before the wipe did not match this repo.** The README described a consumer SS with `GEREGE-WALLET-BFF`; `serverconf` actually held owner `MN/COM/6884857` + subsystem **`EIDMONGOLIA`** publishing two OPENAPI3 descriptions against `https://api.eidmongol.mn/.well-known/openapi/eid-rp/{auth,sign}.yaml`, **both disabled**. `GEREGE-WALLET-BFF` was not present. Trust `serverconf`, not the README.

**Backup before wipe.** `/etc/xroad` complete (incl. `gpghome` private key, `signer/softtoken/` 2 × key p12 + `.softtoken.p12` seal, `ssl/*.p12`, `keyconf.xml`, dead anchor), `/var/lib/xroad` (34 × `.gpg` auto-backups, 45 × `mlog-*.zip` archives), `pg_dump -Fc` of `serverconf`/`messagelog`/`op-monitor`, `ufw status`, `dpkg -l`. `messagelog.logrecord` was empty (0 rows — everything already archived to the zips). The native `backup_xroad_proxy_configuration.sh` refused to run (it encrypts to a GPG key named after the server id); the raw copy replaces it.

**Recovered secrets that this repo redacts:** backup encryption keyid is **`63F7E7C0AF3EF52E`** (`local.ini` → `[proxy] backup-encryption-keyids`), and the per-DB Hibernate passwords are in the backed-up `db.properties`. **Watch out:** a scorched-earth wipe destroys `/etc/xroad/gpghome`, which makes every pre-existing `.gpg` auto-backup permanently undecryptable. Keep the off-host copy of `gpghome` if those backups still matter.

**Wizard answers recovered from debconf** (the 2026-04 note above says the `expect` installer script was lost — this is the data it fed):

```
xroad-common/username             xrd          # NB: debconf said "xrdadmin"; no such user exists.
                                               # The real UI account is xrd (uid 1002, all 5 xroad-* groups).
xroad-common/admin-subject        /CN=ss.gerege.mn
xroad-common/admin-altsubject     IP:66.181.175.134,DNS:ss.gerege.mn
xroad-common/service-subject      /CN=ss.gerege.mn
xroad-common/service-altsubject   IP:66.181.175.134,DNS:ss.gerege.mn
xroad-common/proxy-ui-api-subject /CN=ss.gerege.mn
xroad-common/database-host        (empty → local)
xroad-common/proxy-memory         r
```

**Rebuild.** Purged `xroad-*`, dropped the 3 DBs + their roles, removed `/etc/xroad` `/var/lib/xroad` `/var/log/xroad` `/var/cache/xroad`, then reinstalled **7.8.0**. (cs runs 7.8.2 — patch-level skew between SS and CS is fine, but if you want them aligned, this host is pinned and would need the pin raised.) Package profile: `ee` variant + `xroad-opmonitor` + `xroad-addon-opmonitoring`; the package-shipped `override-securityserver-ee.ini` supplies `key-length=3072` and `enforce-token-pin-policy=true`. All 8 services running. **`4000/tcp Anywhere` stays open** as requested — the router forward `4000 → 10.0.0.27:4000` is live, verified by logging into the UI from the public Internet.

**Configuration was deliberately NOT copied from the old host** (operator instruction: configure fresh, don't cling to the previous setup). The one substantive divergence:

- **Consumer gateway moved to the X-Road standard `8080`/`8443`.** The `ee` variant ships `client-http-port=80` / `client-https-port=443`; the old host kept those and added `connector-host=0.0.0.0` in `local.ini`. Occupying `:80` is what made ACME/HTTP-01 renewal structurally impossible on this host (see the certbot entry below). `local.ini` now sets `8080`/`8443` and keeps `connector-host = 0.0.0.0`, with UFW admitting `8080`/`8443` only from `10.0.0.0/24`. **Watch out:** any information system still calling `ss.gerege.mn:80/r1/...` must be repointed to `:8080`.
- **UFW cleaned of dead-topology rules:** dropped `80/tcp` and `443/tcp` from `38.180.242.76` (the old test.gerege.mn / dbank IS allowances — that topology no longer exists on cs), added `80/tcp Anywhere` for ACME. Also re-worded the `4000/tcp` rule comment, which still read `temp: ministry presentation 2026-05-14` and invited someone to delete a rule that is now a deliberate standing choice. `9100/tcp` from `38.180.242.76` (node_exporter → monitor.x-road.mn) left alone; it is not X-Road config.
- Backup encryption is **not** re-enabled. The old `local.ini` pointed at GPG keyid `63F7E7C0AF3EF52E`, whose private half died with `/etc/xroad/gpghome`. Re-enable against the newly generated key if encrypted auto-backups are wanted.

**Two install gotchas worth keeping:**
- Pinning only the metapackages (`xroad-securityserver-ee=7.8.0-…` etc.) fails — apt resolves the transitive deps to the newest 7.8.2 and dies with `held broken packages`. Pin **all 15** packages explicitly. `/etc/apt/preferences.d/xroad-pin` now holds `Package: xroad-*` / `Pin: version 7.8.0-1.ubuntu24.04` / `Pin-Priority: 1001`, which also stops unattended-upgrades from drifting this host off the instance version.
- Pre-download the debs (`apt-get install --download-only`) *before* purging, and run the whole purge+install under `systemd-run` — an SSH drop mid-`dpkg` leaves a broken package state with no local install source.

**Completed the same day — see the registration entry below.** The text that follows describes the state at the point where the CA was still blocking.

**Configured, with cs admin access, up to the CA step.** Anchor imported → globalconf healthy again; owner + server code + software token initialized; token logged in; TSP added; AUTH + SIGN keys and CSRs generated; the owner and both subsystem clients sit in `SAVED`. What remains is the two CSRs being signed by the approved CA, then import/activate → auth-cert registration → approve on cs → register the two subsystems → approve each. See the instance-rebuild entry below for why the CA is not gerege.mn.

**Access findings:** the `grgdev` operator password works on ss.gerege.mn only (and there `grgdev` has `NOPASSWD: ALL`); it is rejected on cs.xroad.mn and mgmt.xroad.mn, `rp.gerege.mn:22` is unreachable from outside, `grgdev` holds no SSH private key on ss, and the `wg0` management VPN (peer `38.180.145.71:51820`, `10.99.0.0/24`) has had **no handshake in 53 days** — it is dead. cs's admin UI on `:4000` is publicly reachable; its own account is `xrdadmin` (not `xrd`/`grgdev`).

**gerege.mn SSH host key changed** — `~/.ssh/known_hosts:19`, now `SHA256:RxOula4YJ9w+7gCddBj84GieTX6ZFR5GE+PJX8AAZsE`. Benign explanation found: the name no longer points at the host this repo documents. `gerege.mn` now resolves to **38.180.145.75** (repo says `38.180.136.97`), and the CA vhosts `ca.gerege.mn` / `rp-api.eidmongolia.mn` / `eidmongolia.mn` all resolve to **38.180.82.252**. A different machine behind the same name produces exactly this warning, so there is no evidence of interception — but confirm the new fingerprint out-of-band before typing a password at that prompt.

**No access to the CA host.** `xrdadmin` + the supplied passwords get an SSH shell on cs.xroad.mn and mgmt.xroad.mn (on cs it is **not** in sudoers, so `/root/xroad-cs-secrets.txt` and `/root/testca/` are unreadable; on mgmt it has password sudo). None of the three known credential pairs authenticate on `38.180.82.252` (the eID Mongolia CA) or on `gerege.mn`. Signing the two CSRs therefore needs either credentials for the CA host or someone with them running the signing.

## 2026-07-27 — The whole MN instance was re-provisioned on 2026-07-22; every Gerege registration is gone

This is bigger than the anchor. Logging into cs's admin UI (`xrdadmin`) shows a Central Server whose entire contents were created on **2026-07-22** — management requests restart at `id=1`, and the anchor's `generatedAt` is `2026-07-22T11:10:11Z`. cs's IP is unchanged (`38.180.203.234`), so it was **reinstalled in place**, not moved. cs itself now runs **7.8.2**.

What the current instance actually contains, versus what this repo documents:

| | repo (`docs/topology.md`) | cs today |
|---|---|---|
| Member classes | `COM`, `GOV` | `GOV` only (until 2026-07-27, below) |
| Members | 6235972, 6884857, 7181609 (COM), 6806252 (GOV) | `GOV/9900001` "X-Road Operator PoC placeholder", `GOV/5323304` "Үндэсний дата төв" |
| Security servers | rp, ss, paygrid, mgmt | one: `MN:GOV:5323304:mgmt` |
| mgmt.xroad.mn | 38.180.255.177, `GOV/6806252`, `MGMT-XROAD-MN` | **38.180.137.229**, `GOV/5323304`, server code `mgmt` |
| Approved CAs | Gerege Root → Gerege Issuing CA | **eID Mongolia Organization Issuing CA** (`C=MN, O=Gerege Systems LLC`, valid to 2046-07-22, OCSP `https://rp-api.eidmongolia.mn/ocsp`) and **MN X-Road Staging Test CA** (OCSP `http://cs.xroad.mn:8888`) |
| Approved TSA | `https://tsa.timeserver.mn/` | **eID Mongolia TSA** at `http://tsa.timeserver.mn:8318/` |

An `OWNER_CHANGE_REQUEST` (id=6, approved 2026-07-22T18:51) moved mgmt from the PoC placeholder `GOV/9900001` to `GOV/5323304` "Үндэсний дата төв" — so the instance is operated by the National Data Center now, and `mgmt.xroad.mn` is a different host from the one this repo documents.

**Consequences for anyone following the old runbooks:**
- `docs/onboarding-new-member-ss.md` step 2 (`sign-xroad-csr.sh` on gerege.mn) is **wrong now** — the Gerege Root/Issuing CA is not in globalconf, so certificates it signs are rejected. Use the approved CA. Note the new CA is still Gerege's own PKI (`O=Gerege Systems LLC`), just re-issued 2026-07-22 under the eID Mongolia name.
- The TSP URL changed host-port. `https://tsa.timeserver.mn/` refuses connections; it is `http://tsa.timeserver.mn:8318/`.
- rp.gerege.mn and ss.paygrid.mn are unregistered on the new cs, exactly like this host was. They will need the same treatment.

**Registry changes made here (2026-07-27, at operator request).** Added member class `COM` and member `MN/COM/6235972` "Gerege Systems LLC" on cs, so this host could be initialized as its security server. Both propagated into globalconf within ~2 min (cs regenerates on roughly a 10-minute expiry / 1-minute download cycle — `POST /initialization` returns the `init_unregistered_member` warning until it lands, so wait rather than pass `ignore_warnings`).

**State reached on this host:** anchor imported; owner `MN/COM/6235972` + server code `GEREGE-SS-1` + software token initialized and logged in; TSP `eID Mongolia TSA` added; AUTH key `gerege-ss-auth-key-1` (CSR subject `C=MN, CN=GEREGE-SS-1`) and SIGN key `gerege-ss-sign-key-1` (`C=MN, O=COM, CN=6235972`) generated, both CSRs exported; owner and subsystems `EIDMONGOLIA` + `GEREGE-WALLET-BFF` present as clients in `SAVED`. `diagnostics/globalconf` = `OK`.

## 2026-07-27 — `tsa.timeserver.mn` has a bad second A record — timestamping fails ~50% of the time

`diagnostics/timestamping-services` came back `FAIL` with `java.net.ConnectException: Connection refused` even though the TSA was plainly up. Cause: **`tsa.timeserver.mn` resolves to two addresses** —

```
38.180.203.29    :8318 → GET / = 200, POST application/timestamp-query = 200   ← the real TSA
38.180.137.229   :8318 → connection refused                                     ← this is mgmt.xroad.mn
```

Round-robin DNS means roughly every other timestamp attempt hits mgmt.xroad.mn, which does not run the TSA. This affects **every** security server in the instance, not just this one, and it will surface as intermittent message-log timestamping failures rather than a clean outage.

**Real fix:** delete the `38.180.137.229` A record for `tsa.timeserver.mn` (DNS-side; not doable from a security server).

**Stopgap applied on this host:** pinned the good address in `/etc/hosts`:
```
38.180.203.29 tsa.timeserver.mn  # STOPGAP 2026-07-27: DNS also returns 38.180.137.229 (mgmt.xroad.mn) which refuses :8318.
```
**Watch out:** this masks the DNS fault locally and will silently break if the TSA ever legitimately changes address. Remove the line once the bad record is gone.

**Reading the diagnostic afterwards:** with the pin in place the status settles at `WAITING`, not `OK`, and that is correct — X-Road only reports `OK` for a timestamping service once a real message has actually been timestamped, which cannot happen before the server has certificates and traffic. The signals that it is genuinely healthy are (a) no `Connection refused` in `proxy.log`, and (b) the check logging `Timestamp check received HTTP error: 400 - empty request. Might still be ok` — that 400 is the TSA correctly rejecting X-Road's empty probe body; a real `application/timestamp-query` POST to `38.180.203.29:8318` returns 200. Don't "fix" a `WAITING` timestamping service on an unregistered server.

## 2026-07-27 — Restoring the two `EIDMONGOLIA` OPENAPI3 services is blocked upstream

The pre-wipe definitions were recovered by restoring the backed-up `serverconf.dump` into a scratch database, so they are exact rather than reconstructed:

| description URL | service code | service URL | timeout | endpoints |
|---|---|---|---|---|
| `https://api.eidmongol.mn/.well-known/openapi/eid-rp/auth.yaml` | **`auth-sec`** | `https://api.eidmongol.mn/rp/v1` | 60 | `POST /auth/initiate`, `GET /auth/session/*` |
| `https://api.eidmongol.mn/.well-known/openapi/eid-rp/sign.yaml` | `sign-svc` | `https://api.eidmongol.mn/rp/v1` | 60 | `POST /sign/initiate`, `GET /sign/session/*` |

Both were `disabled`, and `serverconf.accessright` was empty — they had no ACL subjects, so nothing could have called them anyway. Note `auth-sec` against `sign-svc`: almost certainly a typo for `auth-svc` (the convention elsewhere), preserved here because the service code is part of the callable path.

**Blocked:** X-Road must download and parse the OpenAPI document to create the description, and the document is not retrievable.

```
POST /clients/MN:COM:6235972:EIDMONGOLIA/service-descriptions
→ 400 {"code":"openapi_parsing_error",
       "metadata":["Error reading OpenAPI description: (handshake_failure) Received fatal alert: handshake_failure"]}
```

`curl` from the host gets **HTTP 526** (Cloudflare "invalid SSL certificate" — the edge cert for `*.eidmongol.mn` is a valid Let's Encrypt one, so the failure is Cloudflare→origin). X-Road's Java client does not even get that far and dies in the TLS handshake. The obvious new-domain equivalents on `rp-api.eidmongolia.mn` return the SPA's 404 HTML, so the documents have not simply moved there.

**To finish this:** fix the origin certificate behind `api.eidmongol.mn`, or supply the current OpenAPI URL, or hand over the two YAML files so they can be served locally and referenced by URL.

## 2026-07-27 — Let's Encrypt cert had expired; fixed by freeing `:80` and switching to standalone

`/etc/letsencrypt/live/ss.gerege.mn/fullchain.pem` had expired **Jul 16 11:42:44 2026 GMT**, with `certbot.service` failing daily on `Some challenges have failed`. Two stacked causes, both structural rather than transient:

1. HTTP-01 needs `:80`, but `:80` was the X-Road client proxy (`ee` variant's `client-http-port=80`), and UFW only admitted `:80` from `38.180.242.76`.
2. The renewal was configured with `authenticator = webroot`, `webroot_path = /var/www/certbot` — but **no web server was serving that directory**, so even a free port would not have validated.

**Fix applied:** moved the consumer gateway to `8080`/`8443` (above), opened `80/tcp` in UFW, and re-issued with `certbot certonly --standalone`, which persists `authenticator = standalone` into the renewal config so the scheduled `certbot.timer` keeps working unattended. New cert valid **2026-07-27 → 2026-10-25**, issuer `Let's Encrypt YE1`, `certbot.service` failure state cleared. DNS-01 is no longer needed.

## 2026-07-27 — `GEREGE-SS-1` fully registered, using **staging** CA certificates

Certificates were issued by the **MN X-Road Staging Test CA** on cs (`/root/testca`, an ordinary `openssl ca` layout: `ca.crt`, `ca.key`, `openssl.cnf`, `index.txt`, `newcerts/`). Root on cs is reachable only as `xrdadmin` → `su root`; `PermitRootLogin prohibit-password` in `/etc/ssh/sshd_config.d/90-xroad-hardening.conf` blocks direct root SSH, and the `sudo` group is empty on both cs and mgmt.

The CA's own `openssl.cnf` already carried the right profiles, so no invention was needed:

| profile | keyUsage | extra |
|---|---|---|
| `[auth_ext]` | `critical,digitalSignature,keyEncipherment` | EKU `clientAuth,serverAuth`, AIA OCSP `http://cs.xroad.mn:8888` |
| `[sign_ext]` | `critical,nonRepudiation` | AIA OCSP |

X-Road classifies a certificate by keyUsage — `digitalSignature` → authentication, `nonRepudiation` → signing — so these two profiles are not interchangeable. The existing mgmt certificates confirmed the subject conventions (`C=MN, CN=<serverCode>` for auth; `C=MN, O=<memberClass>, CN=<memberCode>` for sign), which is exactly what the SS's CSR generator had produced.

```bash
openssl ca -batch -config /root/testca/openssl.cnf -extensions auth_ext -in auth.csr -out auth.cer
openssl ca -batch -config /root/testca/openssl.cnf -extensions sign_ext -in sign.csr -out sign.cer
```

**Two things that must not be skipped:**
- Sign with `openssl ca`, never `openssl x509 -req` — only `openssl ca` records the serial in `index.txt`, and a certificate absent from that index gets an OCSP answer of `unknown`, which a security server treats as invalid.
- The responder is a bare `openssl ocsp -index ... -port 8888` process that reads `index.txt` **at startup only**. It has to be restarted after issuing, or the new serials come back `unknown`. Verified afterwards with `openssl ocsp ... -url http://cs.xroad.mn:8888` → `good` for both.

`openssl ca` writes the human-readable text dump above the PEM block; the certificates were normalised with `openssl x509 -out … -outform PEM` before upload.

**Resulting state.** AUTH serial 1005 (`C=MN, CN=GEREGE-SS-1`), SIGN serial 1006 (`C=MN, O=COM, CN=6235972`), both 2026-07-27 → 2028-07-26, both `active / REGISTERED / OCSP_RESPONSE_GOOD`. Management requests on cs: id 7 `AUTH_CERT_REGISTRATION_REQUEST`, id 8 and 9 `CLIENT_REGISTRATION_REQUEST` — all approved. Owner `MN:COM:6235972` plus subsystems `EIDMONGOLIA` and `GEREGE-WALLET-BFF` all `REGISTERED`. `diagnostics/globalconf` `OK`, `diagnostics/timestamping-services` `OK`, no failed units.

**The approve endpoint is `POST /api/v1/management-requests/{id}/approval`** — not `/approve`, which 404s.

⚠ **These are staging certificates.** The instance's production CA is eID Mongolia Organization Issuing CA (`C=MN, O=Gerege Systems LLC`, OCSP `https://rp-api.eidmongolia.mn/ocsp`), which lives on `38.180.82.252` — no credentials for that host, so nothing was issued there. Re-issue both certificates from it before this server carries anything real; the staging chain is rooted in a test CA whose key sits unprotected in `/root/testca` on the Central Server.

## Watch list for the next operator

- **Staging certificates are the biggest outstanding item.** Re-issue AUTH + SIGN from the production eID Mongolia CA; until then this server's identity chains to a test CA key stored in `/root/testca` on cs.
- **Member ownership changed on 2026-07-27** — this host is Gerege Systems LLC (`MN/COM/6235972`, `GEREGE-SS-1`), not Gerege Core LLC (`6884857`, `CORE-SS-1`). Both are "Gerege" but they are separate legal entities; don't merge them. `README.md`, `docs/topology.md` and the root `README.md` were updated to match.
- **Pre-wipe backup.** Off-host copy at `Enigma/2026-07/ss.gerege.mn-pre-wipe-2026-07-27.tar.gz`; host-side copy kept at `/root/pre-wipe-2026-07-27/` as a safety net until registration completes. It contains live private keys — never commit it to this repo.
- **Member ownership.** This SS is owned by Gerege Core LLC, not Gerege Systems LLC. They are separate legal entities even though both are "Gerege". The member identity in `keyconf.xml` and `serverconf.client` reflects this. Don't merge them.
- **TEST-DEMO is for live demo only** (test.gerege.mn landing page). Don't deprecate it without notice — the demo URL is publicly visible at https://test.gerege.mn and is part of the X-Road launch story.
- **Backups.** Same xroad-proxy daily backup → `/var/lib/xroad/backup/`. GPG backup keyid is REDACTED in the repo; real value in `reference_cs_secrets.md`.
