# OpenControl Server 0.2.2 — administrator guide

[Русский](SERVER_GUIDE_RU.md) | **English**

An IPv4 VPN server for Ubuntu Server 26.04 LTS, amd64 and arm64, interoperating with tested Kerio Control VPN clients.

**OpenControl is an independent project. It is not a Kerio / GFI product and is not affiliated with, sponsored by, endorsed by, or supported by Kerio / GFI.** Names identify interoperability targets only; trademarks belong to their owners. No proprietary client binaries are distributed.

Version 0.2.2 is a prerelease. The unmodified Kerio Control VPN Client 9.6.1 (9306-p1) for Linux was tested on Ubuntu amd64. ARM64 builds and affected tests use QEMU; native ARM hardware and Android acceptance remain pending. Other client versions are not implied to be compatible.

## Capabilities

- Full IPv4 tunnel or split tunnel, pushed routes, DNS servers and search domains.
- TLS control channel, encrypted UDP data, replay protection and rekey through the canonical protocol library.
- Argon2id local accounts; enable/disable, password changes, dynamic/static IPv4 and per-user route/session/timeout policies.
- Owned nftables NAT with automatic outbound interface selection, or external routing without server NAT.
- CLI and root-only JSON helper with CAS revisions and explicit saved/applied outcomes.
- Instance-aware live sessions and durable close events with final traffic counters and cursor/gap reporting.
- JSON journald logs, loopback Prometheus metrics, diagnostics and a hardened dedicated-user systemd service.
- Sanitized client-declared OS name/family/wire code and product version/build in live session responses.

IPv6 bypass must be handled by the operator/client. Client metadata is self-declared and never authorizes access. The admin socket and metrics are local interfaces, not a public HTTP API.

## Download and install

Use Ubuntu Server 26.04 with systemd, `/dev/net/tun` and network/nftables permissions. Containers require operator-provided TUN access and network privileges.

Download the architecture-specific package and SHA256SUMS from [release 0.2.2](https://github.com/nr084/opencontrol-server-releases/releases/tag/server-v0.2.2), or run:

```sh
sudo apt update
sudo apt install ca-certificates curl
arch=$(dpkg --print-architecture)
case "$arch" in amd64|arm64) ;; *) echo 'Unsupported architecture'; exit 1 ;; esac
base=https://github.com/nr084/opencontrol-server-releases/releases/download/server-v0.2.2
curl -fLO "$base/opencontrol-server_0.2.2_${arch}.deb"
curl -fLO "$base/SHA256SUMS"
grep "  opencontrol-server_0.2.2_${arch}.deb$" SHA256SUMS | sha256sum -c -
sudo apt install "./opencontrol-server_0.2.2_${arch}.deb"
```

Continue only after checksum verification succeeds. Checksums verify integrity, not an independent publisher signature. Installation creates service accounts, protected configuration and a self-signed certificate if no certificate/key exists. It does not start the service or enable global forwarding.

## Configure and start

```sh
sudoedit /etc/opencontrol/opencontrol.toml
```

Fresh defaults: TCP/UDP `4090` on `0.0.0.0`, TUN `ocvpn0`, network `10.253.0.0/24`, server `10.253.0.1`, pool `10.253.0.10`–`10.253.0.250`, full IPv4 routing, DNS `1.1.1.1`, NAT enabled with outbound interface `auto`, metrics `127.0.0.1:9108`.

Choose a subnet that does not overlap the server LAN or client networks. TCP and UDP must use the same port. Allow both through the existing firewall and configure router port forwarding where required. Existing firewall FORWARD chains can still block traffic; do not flush unrelated rules.

For split routing, adjust the relevant block:

```toml
[vpn.routing]
mode = "split"
routes = ["192.168.50.0/24"]
```

Provide return routes for internal networks or choose scoped NAT. With `vpn.nat.enabled=false`, provision routing, DNS and fail-closed rules externally. OpenControl manages only its own `inet opencontrol` table. `auto` resolves the real IPv4 outbound route.

```sh
sysctl net.ipv4.ip_forward
sudo sysctl -w net.ipv4.ip_forward=1
sudo opencontrolctl config check
sudo opencontrolctl user add alice
sudo systemctl enable --now opencontrol.service
sudo opencontrolctl status
```

Passwords are entered twice without echo; there is no password argument. For persistent forwarding, create an operator-owned `/etc/sysctl.d/90-opencontrol.conf` containing `net.ipv4.ip_forward=1`. Record the previous value; other applications may require forwarding.

## Client and certificate

Configure the compatible client with the server's public IP/DNS, selected port, username and password. Both TCP and UDP are required.

The generated RSA3072 certificate expires after 365 days; automatic renewal is not provided. Verify its fingerprint through a trusted channel and approve/pin it in the client, or deploy a suitable trusted certificate. Never disable certificate validation; pin formats are client-specific.

```sh
sudo openssl x509 -in /etc/opencontrol/server.crt -noout -fingerprint -sha256
sudo openssl x509 -in /etc/opencontrol/server.crt -noout -dates
```

For your own certificate, use the `[security]` paths, set certificate/key ownership to `root:opencontrol` and mode `0640`, validate configuration and restart. Never publish the private key. After connecting, check sessions, required destinations and client DNS; in full mode also verify the client's external IPv4.

## Users, service and sessions

```sh
sudo opencontrolctl user list
sudo opencontrolctl user show alice
sudo opencontrolctl user add bob
sudo opencontrolctl user add fixed --static-ip 10.253.0.20
sudo opencontrolctl user passwd alice
sudo opencontrolctl user disable alice
sudo opencontrolctl user enable alice
sudo opencontrolctl user delete alice
sudo opencontrolctl --json status
sudo opencontrolctl sessions
sudo opencontrolctl session show 42
sudo opencontrolctl session disconnect 42
sudo opencontrolctl diagnostics
```

Static IPs must pass configuration/uniqueness checks and permit one simultaneous session. User changes are saved atomically and request reload if the daemon is running; a reload failure explicitly reports that the file was saved. Offline changes are supported. Disable/delete/password/static-IP changes close old sessions. Lower limits retain the oldest allowed sessions. A disconnected enabled client may reconnect; disable its account to block access.

```sh
sudo systemctl start opencontrol.service
sudo systemctl stop opencontrol.service
sudo systemctl restart opencontrol.service
sudo systemctl enable opencontrol.service
sudo systemctl disable opencontrol.service
systemctl status opencontrol.service --no-pager
sudo opencontrolctl reload
# alternatively
sudo systemctl reload opencontrol.service
```

`disable` prevents autostart but does not stop an already running service; use `disable --now` for both. Users and log level reload live. Listener, TLS, network, routes, DNS and limits require restart. Validate manual configuration changes before restarting. CLI/systemd commands do not provide the helper's CAS and verified rollback.

Read-only live CLI access requires root or membership in `opencontrol-admin`; reload/disconnect require root. For a trusted observer, use `sudo usermod -aG opencontrol-admin observer`, then refresh that user's login session.

## JSON automation

Run `/usr/sbin/opencontrol-admin-helper` as root with no arguments. Supply one UTF-8 JSON document through stdin and close it; read one JSON response from stdout. Maximum input is 128 KiB; EOF must arrive within 10 seconds. Do not run a web application or the VPN daemon as root.

```sh
printf '%s' '{"api_version":1,"request_id":"version-1","operation":"version"}' | sudo -n /usr/sbin/opencontrol-admin-helper
printf '%s' '{"api_version":1,"request_id":"users-1","operation":"users.list"}' | sudo -n /usr/sbin/opencontrol-admin-helper
printf '%s' '{"api_version":1,"request_id":"ready-1","operation":"readiness"}' | sudo -n /usr/sbin/opencontrol-admin-helper
```

`sudo -n` requires previously authorized noninteractive root access. Never grant untrusted callers generic helper access. Operations: `version`, `users.list/get/create/update/password/delete/apply`, `config.get/validate/apply`, `readiness`, `sessions`, `session.disconnect`, `events.read`. See [API v1](ADMIN_API.md).

Read the current revision, write with `expected_revision`, and inspect `ok`, `saved`, `applied` and `applied_revision`. On `REVISION_CONFLICT`, refresh state. A request ID is correlation, not durable idempotency; do not blindly retry writes after timeout. Even an error can have `saved:true`.

Plaintext passwords for create/password enter stdin only from trusted caller memory, never argv, environment, files, history or logs. Public responses exclude passwords/PHC. Full `users.apply` with validated PHC is trusted root automation only.

Policies: `enabled`, `static_ip`, `max_sessions`, `idle_timeout`, `session_timeout`, `allowed_routes`, `description`. Missing patch fields remain unchanged; null resets optional fields to inheritance (`enabled:null` resets to true). `allowed_routes:[]` restricts traffic to server/DNS exceptions; null inherits server policy. Enforcement updates immediately, advertised routes on reconnect.

Fetch the complete safe config, validate it, then apply with CAS. `restart:false` saves restart-only changes as pending; `restart:true` restarts only the fixed unit and checks new instance, revision, readiness and MainPID. On failure inspect `rollback_verified` and `backup_retained`. The helper has no separate start/stop/enable/disable operations.

## Logs and statistics

```sh
sudo journalctl -u opencontrol.service -f
sudo journalctl -u opencontrol.service --since '1 hour ago' --no-pager
curl -fsS http://127.0.0.1:9108/metrics
sudo opencontrolctl --json sessions
printf '%s' '{"api_version":1,"request_id":"events-1","operation":"events.read","after":0,"limit":100}' | sudo -n /usr/sbin/opencontrol-admin-helper
```

Live sessions expose instance ID, IP, state, age/idle time, RX/TX bytes and packets. Metrics include sessions, authentication, traffic, protocol errors and rejected connections; counters reset on daemon restart. Keep the endpoint on loopback.

Close events include event/instance/session IDs, username, Unix timestamps, duration, reason and final counters. RX is client→server, TX server→client; only inner IP packets count. Responses contain `events`, `cursor`, `oldest_cursor`, `latest_cursor`, `gap`, `healthy`; limit is 1–500.

Store events and cursor in one transaction; deduplicate by node/event ID and identify sessions by node/instance/session ID. Final counters replace the last live snapshot, never add both. Retention is bounded to 4096 events/4 MiB; committed events survive restart. A gap means incomplete history. Do not overwrite/delete the outbox during collection.

Disk failure suspends new admission and makes readiness unhealthy. An unclean crash may lose active sessions' uncommitted counter tails; `previous_shutdown_unclean` makes that explicit. Lossless billing is not guaranteed.

## Client OS metadata

Live sessions add `client_os_family`, `client_os_name`, `client_os_code`, `client_metadata_available`, `client_version`, `client_build`. The tested client reports `linux`, `Ubuntu 26.04`, code `2`, version `9.6.1`, build `9306`. VPN-wire codes 1/2/3 represent Windows/Linux/macOS, not webadmin's enum.

Descriptions are NUL-terminated, sanitized and bounded to 128 UTF-8 bytes. Invalid UTF-8 is safely replaced; empty sanitized text is null. Wire v1/v2 or absent metadata yields unknown/null. Unknown codes retain original code and sanitized name with family unknown. Render supplied text as escaped plain text; none of these fields can establish OS trust or authorize access. API version remains 1; close events are unchanged.

## Upgrade and backup

Protect backups of `/etc/opencontrol`: they contain PHC and private keys. For a consistent config/users/outbox backup, schedule a service stop and also save `/var/lib/opencontrol`. Limit access to trusted administrators; never commit backups to Git.

Download the next architecture-specific package, verify its SHA256 and install using `sudo apt install ./PACKAGE.deb`. Dpkg preserves operator conffiles; compare changes before answering configuration prompts. Upgrades do not automatically restart the daemon; an old process may remain running.

```sh
sudo opencontrolctl config check
sudo systemctl restart opencontrol.service
sudo opencontrolctl --json status
```

Check daemon version, readiness and client traffic. Do not restore an old outbox over newer history: it can roll back cursors and lose events. Automated installers should select eligible `server-v*` entries from the release catalog, accounting for prereleases according to operator policy; `/releases/latest` can omit them.

## Remove

```sh
sudo systemctl disable --now opencontrol.service
sudo apt remove opencontrol-server
# to remove package-managed conffiles too
sudo apt purge opencontrol-server
```

Remove stops the service but retains conffiles. Purge still does not automatically delete generated certificates/keys, lock/rollback files, state journal or service accounts. After backing up what you need and confirming these directories belong only to the removed server, delete explicitly:

```sh
sudo rm -rf -- /etc/opencontrol /var/lib/opencontrol
```

Delete the `opencontrol` user and `opencontrol`/`opencontrol-admin` groups separately only if unused. Do not remove proprietary clients or unrelated networking rules. Remove only firewall/port-forwarding rules and sysctl files you created for this service; restore forwarding only if no other application needs it. Normal shutdown removes owned network resources and does not disable shared forwarding. Downloaded packages/checksum files can be removed if you do not need a local archive.

## Troubleshooting

1. Check systemctl/journald for startup errors, port conflicts, certificate, TUN and nftables issues.
2. Run `sudo opencontrolctl config check` for config/user validation.
3. Confirm both TCP and UDP reach the selected port through firewall/router NAT.
4. Check sessions/diagnostics for authorization, state and assigned IP.
5. For an active session without traffic, check forwarding, DNS, firewall FORWARD, NAT/return routes and allowed routes.
6. For saved but unapplied changes, compare revisions/readiness and restart requirements.

See [diagnostics](https://github.com/nr084/opencontrol-server/blob/v0.2.2/server/DIAGNOSTICS.md), [security](https://github.com/nr084/opencontrol-server/blob/v0.2.2/server/SECURITY.md), and [changelog](https://github.com/nr084/opencontrol-server/blob/v0.2.2/server/CHANGELOG.md).

## ROUTES maintenance (0.2.2)

The server uses canonical opencontrol-protocol 0.2.0. Each complete advertisement
starts and finishes replacement maintenance: flags 3 for split/explicit user routes,
11 for default-covering full mode. Empty lists are valid. Every 600 seconds an
Active session receives the same initial snapshot, including during rekey. This
does not reconnect the client, reset keys or refresh advertised user routes; policy
enforcement still updates live and route advertisement changes require reconnect.
TLS, authentication/MFA handling and the rekey algorithm are unchanged.
