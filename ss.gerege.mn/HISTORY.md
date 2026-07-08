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

## Watch list for the next operator

- **Member ownership.** This SS is owned by Gerege Core LLC, not Gerege Systems LLC. They are separate legal entities even though both are "Gerege". The member identity in `keyconf.xml` and `serverconf.client` reflects this. Don't merge them.
- **TEST-DEMO is for live demo only** (test.gerege.mn landing page). Don't deprecate it without notice — the demo URL is publicly visible at https://test.gerege.mn and is part of the X-Road launch story.
- **Backups.** Same xroad-proxy daily backup → `/var/lib/xroad/backup/`. GPG backup keyid is REDACTED in the repo; real value in `reference_cs_secrets.md`.
