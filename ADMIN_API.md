# Local administration API v1 — OpenControl Server 0.2.2

[Русский](ADMIN_API_RU.md) | **English**

`/usr/sbin/opencontrol-admin-helper` is a one-shot root-only executable. It accepts
one UTF-8 JSON document through stdin and returns one JSON document through stdout.
It takes no arguments, accepts no environment password/path overrides and starts
no listener. Maximum input is 128 KiB; input must reach EOF within 10 seconds.
Core dumps/ptrace dumping are disabled; secret input buffers and PHC values are
cleared. Do not run a web application or the VPN daemon as root.

Fixed paths: `/etc/opencontrol/opencontrol.toml`, `/etc/opencontrol/users.toml`,
`/run/opencontrol/control.sock`. The daemon retains its existing systemd sandbox.
Configuration files must be root-owned and not writable by group/others.

## Requests and responses

Every request requires `api_version: 1`, an ASCII correlation `request_id`
(1–128 letters/digits/`._:-`) and `operation`. Unknown operations and fields,
including nested policy/config fields, are rejected. Correlation is **not durable
idempotency**. Do not blindly retry writes: read the current revision first.

```json
{"api_version":1,"request_id":"example","operation":"users.list"}
```

```json
{"api_version":1,"request_id":"example","ok":true,"saved":true,"applied":true,"revision":"sha256:...","applied_revision":"sha256:...","result":{}}
```

```json
{"api_version":1,"request_id":"example","ok":false,"saved":true,"applied":false,"revision":"sha256:...","applied_revision":"sha256:...","error":{"code":"RELOAD_FAILED","message":"Settings saved but not applied","retryable":true}}
```

Read operations have null `saved`/`applied`. A failed operation may have saved
settings: always inspect both flags. Exit status is 0 for `ok:true`, 1 otherwise.
Errors use stable codes: `INVALID_REQUEST`, `VALIDATION_FAILED`, `NOT_FOUND`,
`ALREADY_EXISTS`, `REVISION_CONFLICT`, `RELOAD_FAILED`, `SERVICE_UNAVAILABLE`,
`PERMISSION_DENIED`, `INTERNAL_ERROR`. Do not parse human-readable messages.
Public responses never include passwords, PHC, raw TOML or TLS key material.

## User operations

| Operation | Additional request fields |
|---|---|
| `users.list` | none |
| `users.get` | `username` |
| `users.create` | `expected_revision`, `username`, `password`, `policy` |
| `users.update` | `expected_revision`, `username`, `patch` |
| `users.password` | `expected_revision`, `username`, `password` |
| `users.delete` | `expected_revision`, `username` |
| `users.apply` | `expected_revision`, `users` (complete registry array) |

Create policy fields: `enabled` (default true), `static_ip`, `max_sessions`,
`idle_timeout`, `session_timeout`, `allowed_routes`, `description`. Optional
fields default to null. Existing username/IP/Argon2/session/route validation is
reused. Static addresses permit one simultaneous session. Username is immutable.

Patch: omitted means unchanged; null resets optional fields to inheritance.
`enabled:null` resets to its default true. `allowed_routes:null` inherits the
server policy; `[]` deliberately restricts traffic to the server/DNS exceptions.
An active client's advertised routes are refreshed on reconnect; enforcement
updates immediately. Disable/delete/password/static-address changes close old
sessions; lower session limits keep the oldest sessions.

Revisions hash the exact global configuration and complete users file, including
PHC changes, with domain/length separation. CAS and read/validate/fsync/rename
use the same `users.lock`. Manual edits already present at CAS are detected.
Cooperating administrative tools must use that lock; an external root editor
can otherwise bypass filesystem coordination.

After saving, helper confirms reload over the Unix API and checks readiness and
the actual applied revision. An unavailable daemon or restart-only pending
configuration leaves `saved:true, applied:false`. Applied revision is not the
last requested revision: it identifies the daemon's accepted registry.

`users.apply` accepts full users with validated Argon2id PHC in `password_hash`.
This is for trusted root-owned automation only. Never expose a generic pass-through
for this operation or grant a web user access to the root registry. All read
responses still exclude PHC. Plaintext passwords enter create/password through
stdin only — never argv, environment, files, shell history or PTY automation.

Example caller pattern (the trusted caller already holds `request` in memory):

```python
completed = subprocess.run(
    ["/usr/sbin/opencontrol-admin-helper"],
    input=json.dumps(request).encode("utf-8"),
    stdout=subprocess.PIPE, stderr=subprocess.PIPE,
    env={}, timeout=120, check=False,
)
response = json.loads(completed.stdout)
```

Do not log the request or include it in exception/audit messages. Protect the
transport to the trusted caller and authorize individual operations there.

## Configuration and readiness

`config.get` returns a safe typed object. `config.validate` accepts that object
as `config` and validates it together with all stored users without writing.
`config.apply` takes `config`, `expected_revision`, and optional `restart:false`.

Safe fields: `listen_address`, one TCP/UDP `port`, `vpn` (interface/network/server
address/pool/routing/DNS/NAT), `limits`, `logging_level`. Fetch config.get rather
than constructing incomplete objects. TLS paths/profile, users/socket paths,
metrics exposure and systemd definitions cannot be changed through this API.

Only user policies and logging reload live. Listener/network/routes/DNS/limits
changes return `result.restart_required:true`. With restart false, they remain
saved and pending. With restart true, only the fixed `opencontrol.service` is
restarted. Success requires matching revision, new instance, readiness and
systemd MainPID. This is not an interface for arbitrary systemctl commands.

Before changing config, helper retains `/etc/opencontrol/opencontrol.toml.rollback`
with protected ownership/permissions. Failed restart attempts a CAS-protected
restore and restart. `result.rollback_verified` reports the checked result;
`backup_retained` identifies retained recovery data. A concurrent newer registry
is never overwritten by rollback. Users and certificate/key files are not reset.

`readiness` returns daemon status; `sessions` returns live sessions.
`session.disconnect` requires both `id` and the original `instance_id` from the
session. Reject stale instances rather than reinterpreting a reused session ID.
Disconnect is asynchronous and a persistent enabled client may reconnect.

Status/diagnostics include `api_version`, `instance_id`, process PID,
`saved_revision`, `applied_revision`, `readiness`, safe running configuration,
`previous_instance_id` and `previous_shutdown_unclean`. Existing status fields
remain. Saved bytes and running configuration can differ while restart is pending.

## Durable close events and monitoring

Daemon-owned `/var/lib/opencontrol/events.json` is an atomic, fsynced bounded
snapshot containing its monotonic cursor. Retention: 4096 close events, maximum
4 MiB. New startup verifies writable storage before reporting ready. Do not delete
or copy over the outbox while collecting history.

`events.read` request: `after` (uint64 cursor), `limit` (1–500). It is available
through the helper and read-only daemon Unix API for root/socket group members.
Response: `events`, `cursor`, `oldest_cursor`, `latest_cursor`, `gap`, `healthy`.
Gap signals retention loss or a cursor beyond this outbox; do not treat it as zero
traffic. Store the cursor transactionally with consumed event IDs.

Close records contain instance/session/event IDs, username, creation/closure time,
duration, final rx/tx bytes and packets, and reason. rx is client→server, tx is
server→client; only inner IP packets count. Persistence precedes session removal
and IP release. Failure freezes the close record, suspends new admission, marks
readiness unhealthy and retries. Failed shutdown persistence produces an error.

Committed close events survive graceful restart and SIGKILL. Counters of an
actively running session remain in RAM: an unclean crash can lose its uncommitted
tail. `previous_shutdown_unclean` makes this explicit; do not invent exact final
totals or promise lossless billing for interrupted active sessions/disk failure.
Final totals replace the last live snapshot, rather than being added twice.

Metrics remain loopback-only and reset with the daemon instance. Use one collector
per node (sessions approximately 5 seconds, metrics 15 seconds); distinguish stale
or unavailable data from zeros. Journald is useful for diagnosis, not a replacement
for the durable cursor API.

## IPv4 routing

Fresh templates default to full mode and NAT; `outbound_interface="auto"` resolves
the real IPv4 route's interface at startup. Forwarding must be explicitly enabled
by the operator. Stop never disables shared forwarding. Review DNS and certificate
trust before starting. Existing operator conffiles are preserved by package upgrades.

For external transparent routing, set NAT false and provision routing/marks/DNS
and fail-closed rules externally. OpenControl touches only its own nft table and
does not bypass user policy. Server is IPv4-only; clients must address IPv6 bypass
themselves. Compatibility is limited to tested client versions/platforms.

## Client metadata (0.2.1)

Live sessions (`sessions`, CLI `session show`, typed helper `sessions`) add
`client_os_family` (`windows`, `linux`, `macos`, `unknown`), `client_os_name`
(string/null), `client_os_code` (raw uint32/null), `client_metadata_available`
(bool), `client_version` (dotted major.minor.patch/null), `client_build` (uint16/null).
VPN-wire codes 1/2/3 map to Windows/Linux/macOS; these are not webadmin enum codes.
For wire v1/v2 or absent metadata all nullable fields are null, family is unknown,
and availability is false. Unknown wire codes retain the raw code and declared
name but family is unknown; no OS is inferred or substituted. Availability denotes
the metadata block's presence, not trust or the validity of its text.

The description is stopped at its first NUL, decoded with UTF-8 replacement,
stripped of control and bidirectional formatting characters and limited to 128
UTF-8 bytes at a character boundary; empty sanitized text is null. Client product
version and build are independent of the OS description. Render all supplied text
as escaped plain text. These fields are self-declared and must never authorize
access. Close events remain unchanged. API version remains 1.
