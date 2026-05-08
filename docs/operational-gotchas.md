# Operational gotchas

A running list of "you'll waste hours debugging this if you don't know" things, organized by symptom.

---

### Symptom: `mlog.no_timestamping_provider_found` from any SS

**Two distinct causes** that look identical:

1. **The SS itself has no TSP entry.** Settings → System Parameters → Timestamping Services → Add → TimeServer.mn. The error originates *on this SS*; nothing further to debug. This is the most common cause for newly-installed member SSes.
2. **The error came back from another SS in the chain.** Look at the stack trace in `/var/log/xroad/proxy.log` — if you see `ClientMessageProcessor.checkResponse` in the trace, the failure was returned from the *peer* (e.g. mgmt.xroad.mn during a `clientReg`). Walk the same checklist on that peer.

### Symptom: `mlog.tsp_certificate_not_found`

The TSA leaf rotated and CS shared-params still references the old leaf cert. Re-add the timestamping service in CS UI with the current `leaf-cert.pem` from `timeserver.mn/tsa-certs/`. Wait 60s for confclient on each member SS to refresh.

### Symptom: `incorrect_validation_info: OCSP response is too old`

The `gerege-ocsp` container served a cached response older than the `freshness` window (3600s default in Mongolia X-Road). Fix:

```bash
ssh gerege.mn 'docker restart gerege-ocsp'
ssh <affected-ss> 'sudo systemctl restart xroad-signer'
```

### Symptom: `Member 'SUBSYSTEM:MN/COM/.../...' has no suitable certificates`

Same root cause as above — OCSP response went stale and the SS dropped the cert from "suitable" set. Same fix.

### Symptom: `Security server has no valid authentication certificate`

The AUTH cert was registered but is not active in `keyconf.xml`. Open SS UI → Keys and Certificates → expand the AUTH key → click the cert → Activate.

### Symptom: `ssl_authentication_failed: Client 'SUBSYSTEM:.../...' has no IS certificates`

The producer subsystem on rp.gerege.mn has its OpenAPI3 service URL set to `https://...` AND no IS TLS certificate uploaded. Either:
- Upload the IS server's TLS cert under Internal Servers → Information System TLS certificate (recommended for self-signed IS), OR
- Set the OpenAPI server URL scheme to `http`, OR
- Switch connection type to HTTPS_NOAUTH (only applies to consumer role though).

### Symptom: TLS handshake fails when registering a new member SS, even though everything looks right

Walk through ALL of:
1. CS UFW allows the new SS's IP on 4001 + 4002.
2. OCSP responder is fresh (`docker restart gerege-ocsp` then restart xroad-signer everywhere that matters).
3. The OCSP cert AIA URL in the partner's auth cert ends in `/ocsp` (legacy certs without `/ocsp` rely on the nginx root POST rewrite at ocsp.gerege.mn).
4. The new SS has TimeServer.mn as a TSP entry.
5. The mgmt SS has all 4 prerequisites (TSP, WSDL, IS cert, ACL — see `mgmt.xroad.mn/README.md`).

### Symptom: Cyrillic national_id (`МА...`) lookups fail through X-Road but work via direct backend curl

X-Road producer SS forwards URL paths URL-encoded (`%d0%9c%d0%90...`). Fiber's `c.Params()` does NOT auto-decode path segments. The handler must call `url.PathUnescape(c.Params(...))` explicitly. Already patched in `eid-gerege-backend/internal/handler/rp/routes.go::certLookup`.

### Symptom: Sigstore TSA refuses to start with `panic: certificate must have extended key usage timestamping set to sign timestamping certificates`

The `certchain.pem` it loads contains an intermediate without `id-kp-timeStamping` EKU. The Gerege Issuing CA does NOT have this EKU and therefore can't be used to sign the TSA leaf chain. Use the dedicated `Gerege TSA Issuing CA` (under `gerege.mn/xroad-ca/tsa-issuing/`) which carries the EKU. See `timeserver.mn/README.md` for the rotation procedure.

### Symptom: nginx test passes but `conflicting server name "X" on 0.0.0.0:443, ignored` warnings

Some `.bak` of a server block is still in `sites-enabled/`. Move it to `sites-available/` or delete. Otherwise nginx may serve the older block first and the new `location /xroad/v1/` rules silently never apply.
