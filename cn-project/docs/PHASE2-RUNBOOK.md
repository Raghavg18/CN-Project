# Phase 2 setup and demonstration runbook

Run this from the `cn-project` directory on each Mac. Start from working Phase 1. [Architecture and IP inventory](PHASE2-ARCHITECTURE.md) · [Evidence checklist](../evidence/phase2/README.md)

| Variable | Current value |
| --- | --- |
| Primary DNS, Mac 1 | `10.7.15.27` |
| Primary edge and backup DNS, Mac 2 | `10.7.3.98` |
| Backend and standby edge, Mac 3 | `10.7.21.8` |
| LAN interface | `en0` |

Recheck each IP with `ipconfig getifaddr en0`. If a lease changes, update the bundled hosts and client resolver files, the relevant config inputs below, and the source IP or interface in `phase2/firewall/backend.pf`. Record the updated topology before running the demos. Mac 3 needs administrator access for the firewall and standby edge.

## A. Add the backup DNS resolver

**Deploy on Mac 2 first.** If the backup is already staged, see [the staged config and validation](../evidence/phase2/03-backup-staged.txt), then run `sudo "$(command -v brew)" services start dnsmasq`. Otherwise, use the block below with Mac 2's current address as `DNS_IP`. The template binds to loopback and that Mac's LAN IPv4 address. Put both resolvers on the clients before restarting or stopping Mac 1's DNS.

```sh
brew list dnsmasq >/dev/null 2>&1 || brew install dnsmasq
BREW_PREFIX="$(brew --prefix)"
BREW="$(command -v brew)"
DNS_IP=10.7.3.98  # On Mac 1 use 10.7.15.27.
DNS_CONF="$BREW_PREFIX/etc/dnsmasq.conf"
LIVE_HOSTS="$BREW_PREFIX/etc/dnsmasq.solo.hosts"
DNS_BACKUP="$(mktemp -d "$HOME/solo-phase2-dns.XXXXXX")"
printf 'DNS backup saved in %s\n' "$DNS_BACKUP"
sudo cp -p "$DNS_CONF" "$DNS_BACKUP/dnsmasq.conf"
if [ -f "$LIVE_HOSTS" ]; then sudo cp -p "$LIVE_HOSTS" "$DNS_BACKUP/dnsmasq.solo.hosts"; fi

sed -e "s|@BREW_PREFIX@|$BREW_PREFIX|g" -e "s|@DNS_IP@|$DNS_IP|g" \
  phase2/dns/dnsmasq.conf.template > "$DNS_BACKUP/phase2.conf"
"$BREW_PREFIX/sbin/dnsmasq" --test --conf-file="$DNS_BACKUP/phase2.conf" &&
  sudo install -m 644 phase2/dns/primary.hosts "$LIVE_HOSTS" &&
  sudo install -m 644 "$DNS_BACKUP/phase2.conf" "$DNS_CONF" &&
  sudo "$BREW" services restart dnsmasq
```

If Homebrew says root cannot download metadata, use `brew update` as your normal user, then repeat only the service restart. Reuse the printed backup directory for rollback. To restore Mac 1's configuration, copy its saved `dnsmasq.conf` back and restart the service. Also restore its saved hosts file if present. To remove the new backup service from Mac 2, stop it with `sudo "$(command -v brew)" services stop dnsmasq`, then restore Mac 2's saved configuration and any saved hosts file.

From a **different** Mac, verify both resolvers return the same IP for both names. Mac 1 may still return the Phase 1 TTL of 60 at this point. Then configure the client Macs (Macs 2 and 3) to list both addresses for `solo.test`. This extends the existing private resolver file and preserves automatic Wi-Fi DNS for public names.

```sh
for dns in 10.7.15.27 10.7.3.98; do
  dig @"$dns" app.solo.test A +time=2 +tries=1 +noall +answer
  dig @"$dns" api.solo.test A +time=2 +tries=1 +noall +answer
done
sudo mkdir -p /etc/resolver
if [ -f /etc/resolver/solo.test ]; then
  sudo cp -p /etc/resolver/solo.test "/private/tmp/solo.test.phase1.$(date +%Y%m%d-%H%M%S)"
fi
sudo install -m 644 phase2/dns/solo.test.resolver /etc/resolver/solo.test
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

Verify normal project HTTPS from both clients before proceeding. The failover test works while Mac 1 still publishes its Phase 1 TTL because both resolvers return the same names and addresses.

On Mac 1 stop only dnsmasq with `sudo "$(command -v brew)" services stop dnsmasq`. On Mac 3, flush the client DNS cache, then show that a direct query to Mac 1 times out and a **normal** `curl -fsS https://app.solo.test/api/status` still succeeds. Start Wireshark on Mac 3 before the normal curl and use `dns.qry.name == "app.solo.test"` to capture the backup reply from Mac 2. Direct `dig @10.7.3.98 app.solo.test` is a useful cross-check. Restart Mac 1 with `sudo "$(command -v brew)" services start dnsmasq` and verify it answers again. Record the stop, normal request, backup reply and restore in A evidence. To roll back the client change, restore its saved `solo.test.phase1.*` file (or remove the new file if there was no prior rule), then flush the cache again.

Before B and E, repeat the deployment block on Mac 1 with `DNS_IP=10.7.15.27`. Verify both resolvers now return TTL 30. This prepares the shared record reload mechanism for the timed cutover.

## Prepare both nginx edges for B, D and E

**On Mac 2**, use the existing trusted cert/key under `$(brew --prefix)/etc/nginx/certs/solo.{crt,key}`. **On Mac 3**, install nginx and OpenSSL 3 if needed with `brew install nginx openssl@3`. From `cn-project` on Mac 3, make a private key and CSR:

```sh
mkdir -p "$HOME/cn-standby-tls"
chmod 700 "$HOME/cn-standby-tls"
OPENSSL="$(brew --prefix openssl@3)/bin/openssl"
(umask 077; "$OPENSSL" req -new -newkey rsa:2048 -nodes \
  -keyout "$HOME/cn-standby-tls/standby.key" \
  -out "$HOME/cn-standby-tls/standby.csr" -config certs/leaf.cnf)
```

Generate the key once. Reuse it if you already have a signed leaf. Transfer the CSR to Mac 2 as `~/Downloads/standby.csr`. From `cn-project` on Mac 2, sign it with the existing CA and verify both SANs:

```sh
OPENSSL="$(brew --prefix openssl@3)/bin/openssl"
"$OPENSSL" x509 -req -in "$HOME/Downloads/standby.csr" \
  -CA "$HOME/cn-certs/rootCA.pem" -CAkey "$HOME/cn-certs/rootCA.key" \
  -CAcreateserial -out "$HOME/Downloads/standby.crt" -days 397 -sha256 \
  -extfile certs/leaf.cnf -extensions v3_leaf &&
  "$OPENSSL" verify -CAfile "$HOME/cn-certs/rootCA.pem" "$HOME/Downloads/standby.crt" &&
  "$OPENSSL" x509 -in "$HOME/Downloads/standby.crt" -noout -ext subjectAltName
```

Transfer the signed `standby.crt` to Mac 3's Downloads. Keep both private keys on their respective machines. Install the new leaf and local key **on Mac 3 only**:

```sh
BREW_PREFIX="$(brew --prefix)"
sudo mkdir -p "$BREW_PREFIX/etc/nginx/certs"
sudo install -m 644 "$HOME/Downloads/standby.crt" "$BREW_PREFIX/etc/nginx/certs/solo.crt"
sudo install -m 600 "$HOME/cn-standby-tls/standby.key" "$BREW_PREFIX/etc/nginx/certs/solo.key"
```

Install the shared edge template separately on Macs 2 and 3. Use `primary` / `10.7.21.8` on Mac 2 and `standby` / `127.0.0.1` on Mac 3. Keep the backup outside `nginx/servers/`, where Homebrew would otherwise load it as another server config.

```sh
BREW_PREFIX="$(brew --prefix)"
BREW="$(command -v brew)"
EDGE_ID=primary       # On Mac 3 use standby.
UPSTREAM_IP=10.7.21.8 # On Mac 3 use 127.0.0.1.
NGINX_CONF="$BREW_PREFIX/etc/nginx/servers/solo.conf"
NGINX_BACKUP="$(mktemp -d "$HOME/solo-phase2-nginx.XXXXXX")"
printf 'nginx backup saved in %s\n' "$NGINX_BACKUP"
if [ -f "$NGINX_CONF" ]; then sudo cp -p "$NGINX_CONF" "$NGINX_BACKUP/solo.conf"; fi
sed -e "s|@BREW_PREFIX@|$BREW_PREFIX|g" -e "s|@EDGE_ID@|$EDGE_ID|g" \
  -e "s|@UPSTREAM_IP@|$UPSTREAM_IP|g" phase2/nginx/solo.conf.template \
  > "$NGINX_BACKUP/phase2.conf"
sudo install -m 644 "$NGINX_BACKUP/phase2.conf" "$NGINX_CONF" &&
  sudo "$BREW_PREFIX/bin/nginx" -t && sudo "$BREW" services restart nginx
```

If nginx validation fails, restore the backup before restarting. On a newly configured Mac 3, move the new `solo.conf` out of `servers/` to roll back. Confirm trusted HTTPS on each edge with `curl --resolve app.solo.test:443:10.7.3.98 -fsSI https://app.solo.test/api/status` and repeat with `10.7.21.8`. Check for `X-Edge: primary` and `X-Edge: standby`. Keeping the hostname in the URL preserves SNI and certificate validation during these checks.

## C. Isolate the backend ports

On Mac 3, keep both Node processes running. Back up the active rules for reference, and use a dedicated temporary anchor under the installed macOS filter hook. If the live main ruleset lacks the `com.apple/*` filter hook or the demo anchor already has rules, stop and use the faculty-provided rule template or agree on a clean anchor first.

```sh
PF_BACKUP="$(mktemp -d "$HOME/solo-phase2-pf.XXXXXX")"
sudo pfctl -sr | tee "$PF_BACKUP/main.rules"
sudo pfctl -a com.apple/solo-phase2 -sr | tee "$PF_BACKUP/anchor.before"
sudo pfctl -a com.apple/solo-phase2 -nf phase2/firewall/backend.pf
```

Confirm the hook is present and the anchor is empty, then load the rules and hold a PF enable reference. Save the token printed by `pfctl -E` for rollback.

```sh
sudo pfctl -a com.apple/solo-phase2 -f phase2/firewall/backend.pf &&
  sudo pfctl -E 2>&1 | tee "$PF_BACKUP/enable.txt"
sudo pfctl -a com.apple/solo-phase2 -vvsr
```

Open fresh connections from Mac 2 and Mac 1 using `curl --connect-timeout 3 --max-time 5 -i http://10.7.21.8:3001/api/status` and repeat on `3002`. Expect both to work from Mac 2 and both to fail from Mac 1. From Mac 1, confirm `curl --resolve app.solo.test:443:10.7.3.98 -fsSI https://app.solo.test/api/status` still returns `200` through nginx. Capture the curl results and Mac 3's rule counters. Then remove only the temporary rules and release only this demo's enable reference:

```sh
sudo pfctl -a com.apple/solo-phase2 -F rules
sudo pfctl -X YOUR_ENABLE_TOKEN
sudo pfctl -a com.apple/solo-phase2 -sr
```

Repeat the direct checks from Mac 1 and confirm both ports are reachable again. Mac 3 hosts the services and is unsuitable for the independent blocked-client test.

## D. Prove backend process failover

From a client, run six requests over HTTPS using the name and save `X-Edge` and `X-Backend` headers. On Mac 3 stop **only A** with Ctrl+C in its own backend tab. Repeat the six requests and show `200` with `X-Backend: B` throughout. Restart A with `BACKEND=A PORT=3001 node server.js` from `~/cn-backend` or `npm run start:a` from the repo's `backend/`. Wait up to the configured 10-second passive failure window and repeat to show both IDs again.

```sh
for i in 1 2 3 4 5 6; do
  /usr/bin/curl -fsS -D - -o /dev/null https://app.solo.test/api/status |
    grep -iE '^(HTTP/|x-edge:|x-backend:)'
done
```

Include the single failed upstream log line if available. The primary edge is the sole active ingress in this test. Automatic edge failover would require an independently monitored front door or health-driven routing with its own failure handling. The assignment's Extension D stops the A process. If faculty asks to power off Mac 3, agree on a layout with B on a different Mac first.

## B and E. Demonstrate TTL and controlled edge cutover

Keep both edges running. On each DNS Mac, locate the installed launchd label before the timed exercise. Homebrew versions use `sh.brew.dnsmasq` or `homebrew.mxcl.dnsmasq`. Print the loaded job to verify the label. Reauthorize `sudo` in the working terminals before timing. `SIGHUP` reloads the `addn-hosts` file without a full service restart.

```sh
BREW_PREFIX="$(brew --prefix)"
DNS_LABEL=sh.brew.dnsmasq  # Use the label actually loaded on this Mac.
sudo launchctl print "system/$DNS_LABEL" | head -n 12
sudo -v
```

1. On Mac 3, clear the client cache, then warm `app.solo.test` through a normal HTTPS request. Save its timestamp and `X-Edge: primary`. Use a direct DNS query to record TTL 30. Have both DNS operators ready before warming the name.
2. Immediately run the following on **both DNS Macs**. The only changed DNS record is `app.solo.test`.

   ```sh
   sudo install -m 644 phase2/dns/standby.hosts "$BREW_PREFIX/etc/dnsmasq.solo.hosts" &&
     sudo launchctl kill SIGHUP "system/$DNS_LABEL"
   ```

3. Compare fresh `dig @10.7.15.27 app.solo.test A +noall +answer` and `dig @10.7.3.98 app.solo.test A +noall +answer` with repeated normal `curl -fsSI https://app.solo.test/api/status` from the warmed client. Capture the old edge if the cached answer is still valid, then capture `X-Edge: standby` after expiry. If the cache window elapsed before both updates, roll back and repeat with the operators ready. Check that `api.solo.test` still shows `X-Edge: primary`.
4. Roll back both resolvers using the same command with `phase2/dns/primary.hosts`. Clear the client cache and confirm the primary response. Repeat the cutover for a separate trial, flush with `sudo dscacheutil -flushcache` and `sudo killall -HUP mDNSResponder`, and capture the immediate switch to standby without waiting for expiry. Restore the primary hosts file on both resolvers and confirm both answers and the normal HTTPS path are back on Mac 2.

## F. Diagnose a faculty-injected fault

Work through DNS (`dscacheutil` or a direct query), TCP (`nc -vz` to the resolved IPv4 and port 443), TLS (`curl -v` by hostname with verification), then application status and `X-Edge` / `X-Backend`. Explain each result before moving down the stack. Save the fault, first failing layer, fix or isolation, and proof after recovery. Any team rehearsal should be labeled as such in the evidence.

## Final live demonstration

Use the assignment's order. (1) Show topology and inventory. (2) Verify LAN reachability. (3) Resolve the project domain. (4) Open trusted HTTPS. (5) Show A/B requests. (6) Explain DNS, TCP and TLS in the saved capture. (7) Show cache headers and a conditional `304` or cache hit. (8) Stop A and show B-only success. (9) Demonstrate DNS resilience and keep all A/B/E evidence ready. (10) Diagnose the faculty fault. (11) Answer individual viva questions. Keep the firewall evidence, rollback, and the [report](PHASE2-REPORT.md) available for review.
