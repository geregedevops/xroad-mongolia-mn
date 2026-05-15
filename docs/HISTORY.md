# docs/ — operational history

## 2026-04-19 — Initial monorepo creation

Per-server folders created, configs gathered, READMEs written, secrets sanitized to `REDACTED`. Cross-cutting docs (`topology.md`, `pki-architecture.md`, `onboarding-new-member-ss.md`, `operational-gotchas.md`) authored from the running production state and the operator's local memory.

## What this folder is for going forward

- `topology.md` — keep updated whenever a new SS is added, ports change, or an IP rotates.
- `pki-architecture.md` — update when CA hierarchy changes (new intermediate, key rotation, EKU profile addition).
- `onboarding-new-member-ss.md` — whenever the partner-onboarding playbook gets new steps (e.g. new mandatory firewall rule), append here. Future partners + their consultants will rely on this being current.
- `operational-gotchas.md` — every time a new "I burned 2 hours debugging this" incident happens, append the symptom + cause + fix here. The point is to never burn that 2 hours twice.

## 2026-04-19 — All 6 servers switched to Asia/Ulaanbaatar timezone

`cs.gerege.mn`, `mgmt.gerege.mn`, `rp.gerege.mn`, `ss.gerege.mn`, `gerege.mn`, `timeserver.mn` were all running on UTC (display only — system clocks were NTP-synced and correct). Switched to `Asia/Ulaanbaatar` (+08:00) so log lines and CLI tools render in the legal local time of the operators.

```bash
sudo timedatectl set-timezone Asia/Ulaanbaatar
```

Then restarted the X-Road services (`xroad-confclient`, `xroad-signer`, `xroad-proxy`, `xroad-center` where applicable), the `eid-gerege-backend` + `gerege-ocsp` containers, and Sigstore TSA (`timestamp-authority`) so each picks up the new TZ for its log output.

**Watch out:** Protocol-level timestamps remain UTC — RFC 3161 TSA tokens, OCSP `thisUpdate`/`nextUpdate`, X-Road message timestamps are all UTC by spec. The TZ change is purely cosmetic / operator-friendliness; signing, OCSP freshness, and X-Road clock-skew validation are unaffected. NTP sync continues uninterrupted.

The `eid-gerege-backend` Go service writes log timestamps as Unix epoch milliseconds (TZ-agnostic) via zerolog. Switching the human-readable display would require a separate `TimestampFieldName` config change — left for a future cleanup.

## Watch list for the next maintainer

- The HISTORY.md files in each per-server folder capture chronological incidents. The cross-cutting `operational-gotchas.md` here de-duplicates by symptom — it's the first place an operator should look when something breaks. Keep both in sync.
- Diagrams across all docs are Mermaid — GitHub renders them natively in fenced ```` ```mermaid ```` blocks ([GitHub docs](https://docs.github.com/en/get-started/writing-on-github/working-with-advanced-formatting/creating-diagrams)), and the `x-road.mn/` landing page proves the same syntax round-trips through `mmdc` for the public site. Policy reversed 2026-05-14 — the earlier "ASCII only" rule was retired so the Mongolia X-Road monorepo can ship a single visual language across docs + landing page + slide decks. When editing Mermaid, keep the source diff-readable: one node/edge per line, `%%` comments for intent, prefer `sequenceDiagram autonumber` for time-ordered flows and `graph LR/TD` for topology.
