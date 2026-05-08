# Onboarding a new member Security Server

Use this checklist when a partner (bank, GovTech agency, fintech) wants to consume Mongolian X-Road services or publish their own.

## 0. What you need from the partner

- Legal entity name + member class (`COM` / `GOV` / `NEE` / etc) + state register number (e.g. `1234567`).
- The public IP of their Security Server.
- Whether they will be a CONSUMER, PRODUCER, or BOTH.
- Their preferred subsystem code(s), e.g. `BANK-LOAN-APP`, `GOV-TAX-FILING`.

## 1. CS-side member registration (you, on cs.gerege.mn)

1. Open CS UI: `ssh -L 14000:localhost:4000 cs.gerege.mn`, then https://localhost:14000.
2. Members → Add Member: name + class + code (the state register number).
3. Don't create the SS row yet — that comes from the partner's `clientReg` request (see step 4).

## 2. CA-side cert issuance for the partner's SS

The partner generates AUTH and SIGN keys on their SS in their X-Road UI → Keys and Certificates → Generate key. They produce two CSRs and send them to you.

On gerege.mn, sign each CSR with the matching profile:

```bash
ssh gerege.mn
cd /opt/xroad-ca
sudo ./sign-xroad-csr.sh /tmp/partner-auth.csr auth   # → partner-auth.auth.cer
sudo ./sign-xroad-csr.sh /tmp/partner-sign.csr sign   # → partner-sign.sign.cer
```

Send the two `.cer` files back to the partner. They import via X-Road UI → Keys and Certificates → Import certificate, then activate each.

## 3. Network plumbing

- CS UFW: `sudo ufw allow from <PARTNER_IP> to any port 4001 proto tcp`  (globalconf)
- CS UFW: `sudo ufw allow from <PARTNER_IP> to any port 4002 proto tcp`  (mgmt service)
- If the partner is a CONSUMER and will call rp.gerege.mn: nothing extra (rp.gerege.mn:5500 is public).
- If the partner is a PRODUCER and you (or another consumer) will call them: nothing on our side; they configure their own SS.

## 4. Partner registers their SS at CS

Partner workflow on their SS UI:
1. Configuration anchor — download from `http://cs.xroad.mn/internalconf?version=2` (or copy `mgmt.xroad.mn/xroad/configuration-anchor.xml` from this repo).
2. Generate AUTH + SIGN keys, produce CSRs (see step 2 above for signing).
3. After importing both certs and activating them, fill **Initial Configuration** wizard with their member identity + SS code.
4. **Add TSP entry** (Settings → System Parameters → Timestamping Services → Add → TimeServer.mn at `https://tsa.timeserver.mn/`). **Without this, every clientReg fails with `mlog.no_timestamping_provider_found`.**
5. UI now shows their owner client in SAVED state — click Register. This sends `clientReg` over X-Road msg via mgmt SS, which proxies to CS.

## 5. CS-side request approval

Their clientReg lands in CS UI → Management Requests as Pending. Approve. Their owner client flips to REGISTERED.

## 6. Subsystems

For each subsystem the partner wants:
1. Partner UI → Clients → owner → Add subsystem → `<CODE>` → Register.
2. CS UI → Management Requests → Approve.

## 7. As a CONSUMER of GEREGE-ID services

Once registered, the partner can call:
```
POST/GET https://<their-ss>/r1/MN/COM/6235972/GEREGE-ID/{auth,sign,cert}-svc/<openapi-path>
Header: X-Road-Client: MN/<class>/<code>/<subsystem>
```

But it will fail with `access_denied` until WE grant their subsystem access on rp.gerege.mn. To grant:

1. SSH-tunnel to rp UI: `ssh -L 4000:localhost:4000 rp.gerege.mn`.
2. https://localhost:4000 → Clients → GEREGE-ID → Service clients → Add subjects → `MN/<class>/<code>/<subsystem>`.
3. Tick the operations they should be able to call (auth-svc, sign-svc, cert-svc).
4. Save.

That's it. **There is no backend DB step.** As of 2026-04-19 the eid-gerege-backend has exactly one `relying_parties` row — the X-Road Gateway (`00000001-0000-4000-8000-000000000000`) — and trusts ANY caller that arrives via the /xroad/v1 trust gate (nginx IP-pinned to rp.gerege.mn + `X-Gerege-SS-Token`). The `X-Road-Client` header value is captured into `sessions.xroad_client` and `audit_logs.details` for traceability.

In other words: rp.gerege.mn's Service-clients ACL is the single source of truth for "who can call what". No SQL on the backend side per partner.

End-to-end is now live for that partner.

## 8. As a PRODUCER

The partner publishes their own services on their SS via Services → Add REST/WSDL. We don't need to do anything on our infrastructure unless we, or another member, want to consume them — in which case follow step 7 with our subsystem identifier.
