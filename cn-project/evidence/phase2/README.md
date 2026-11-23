# Phase 2 evidence index

The assignment accepts terminal output or screenshots. Label each capture with the Mac, time, commands and result. Wireshark files should contain only the project traffic needed for the demonstration. Keep private keys and credentials outside the repository.

## Collected evidence

| Evidence | What it proves |
| --- | --- |
| [01-baseline.txt](01-baseline.txt) | Live Phase 1 entry baseline from Mac 2. Abhi answers both names, Kevish reaches nginx by hostname, A/B alternates, and public HTTPS works with automatic Wi-Fi DNS. |
| [02-isolated-lab.txt](02-isolated-lab.txt) | Local-only validation of the Phase 2 configs and behavior. The lab uses temporary ports and TLS certificates. |
| [03-backup-staged.txt](03-backup-staged.txt) | Mac 2's installed backup DNS config and successful syntax check. This records staging, before activation. |
| [04-backup-live.txt](04-backup-live.txt) | Mac 2's running service and both private names over UDP and TCP with TTL 30. Normal project and public HTTPS both returned `200`. |
| [05-backup-lan.txt](05-backup-lan.txt) | Operator-supplied replies from the Mac 3 LAN check. Both names reached the backup and returned the expected IP and TTL. |
| [06-client-scope.txt](06-client-scope.txt) | Both clients list the two project resolvers and reach the app by its normal hostname. Mac 2 also shows the active macOS scope and public HTTPS with automatic Wi-Fi DNS. |
| [A-backup-dns.pcapng](A-backup-dns.pcapng), [Wireshark screenshot](07-backup-dns-wireshark.png), [packet and application observations](07-backup-dns-observations.txt) | Mac 3's query and reply from backup DNS with TTL 30. Mac 2 recorded the primary timeout, successful HTTPS and primary recovery. |
| [08-matching-dns-ttl.txt](08-matching-dns-ttl.txt) | Both live resolvers return `10.7.3.98` with TTL 30 for `app` and `api` before cutover. |
| [09-standby-tls-issued.txt](09-standby-tls-issued.txt) | The standby certificate was signed by the project CA, matches Raghav's CSR, and passes TLS-server validation for both hostnames. |
| [10-standby-https-live.txt](10-standby-https-live.txt) | Both hostnames return trusted HTTPS and serve A and B through Raghav's standby. The normal project hostname still reached the existing primary during this check. |
| [11-cutover-ready.txt](11-cutover-ready.txt) | Both live edges return trusted HTTPS with their expected identity headers. Both resolvers still point `app.solo.test` to the primary with TTL 30. |
| [12-cutover-first-trial.txt](12-cutover-first-trial.txt) | First watch run and follow-up. Both resolvers served the primary throughout this trial. The successful retry is recorded below. |
| [13-live-cutover.txt](13-live-cutover.txt) | The successful timed cutover. Both fresh answers moved to standby before macOS switched the normal HTTPS request. The API control stayed on the primary. |
| [14-rollback-and-flush.txt](14-rollback-and-flush.txt) | Raghav reached standby before rollback and primary after the flush. Mac 2 verified both resolvers and both hostnames were back on the primary. |
| [15-firewall-baseline.txt](15-firewall-baseline.txt) | Abhi and Kevish could both reach backend ports `3001` and `3002` before applying the temporary firewall rules. |
| [16-firewall-active.txt](16-firewall-active.txt) | Active PF rules, both direct connections blocked from Abhi, both allowed from Kevish, and HTTPS through the edge still working. |
| [17-firewall-rollback.txt](17-firewall-rollback.txt) | All four PF rules saw traffic. The demo rules were cleared and Abhi could reach both backend ports again. |
| [18-backend-failover-baseline.txt](18-backend-failover-baseline.txt) | Six healthy HTTPS requests alternated A and B on the primary before the process-stop test. |
| [19-backend-a-stopped.txt](19-backend-a-stopped.txt), [20-backend-recovered.txt](20-backend-recovered.txt) | Six B-only HTTPS `200` responses while A was stopped, nginx's upstream connection refusal, then successful responses from both backends after A returned. |
| [21-troubleshooting-rehearsal.txt](21-troubleshooting-rehearsal.txt) | Team rehearsal. DNS resolved, the wrong TCP port refused the connection, and HTTPS recovered on port 443. |
| [22-final-demo-preflight.txt](22-final-demo-preflight.txt) | Both DNS servers and trusted HTTPS edges, primary A/B alternation, HTTP `304`, and the two captures for the live demo. |

## Live evidence still to collect

The following are suggested filenames to use when each test is actually completed. The commands and rollback are in the [Phase 2 runbook](../../docs/PHASE2-RUNBOOK.md).

| Extension | File | Capture |
| --- | --- | --- |
| A | `A-backup-dns.txt` | Add Mac 3's paired terminal transcript if available. The raw capture and independent checks are already indexed above. |
| B | `B-dns-ttl.txt` | Optional stronger live flush evidence. Capture the cached edge after rollback and immediately before the flush to isolate the flush from natural expiry. |
| F | `F-troubleshooting.txt` | Faculty-injected fault, observations from DNS through the first failing layer, diagnosis or fix and retest. Label any earlier rehearsal separately. |

The original [Phase 1 evidence index](../README.md) covers the protocol capture, HTTP cache validation and the five Phase 1 failure demonstrations. For final review, have the [topology](../../docs/PHASE2-ARCHITECTURE.md), [configs](../../phase2/), [backend source](../../backend/), and [Phase 2 report](../../docs/PHASE2-REPORT.md) ready alongside this folder.
