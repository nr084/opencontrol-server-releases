# Changelog

[Русский](CHANGELOG_RU.md) | **English**

## 0.9.0 — 2026-10-26

- Pin Protocol Library =0.9.0 / cacf1a738e5013922ca9678d529d014fffc22d11 with the legacy build feature and runtime LegacyPolicy.
- Delegate session epochs, key generations, ACK barriers, same-slot peer replacement and independent liveness to ProtocolSession; bounded transport close drops key/binding state.
- Persist security.allow_legacy (default false), expose limits.allow_legacy in API v1 and reload live. Revoking permission closes sessions retaining any legacy key generation; AES-only sessions continue.
- Refuse disabled legacy through typed INVALID_REQUEST, without numeric local error enums, downgrade or TLS ciphersuite selection.
- Use canonical advertisement_batches for full/empty/segmented/repeated ROUTES. Never suppress retransmission on Unchanged; safeLan remains status, not firewall policy.
- Separate ordinary server/client profile tests from S2S initiator/acceptor contract tests; S2S is not an enabled product listener.
- Report UDP cipher/integrity separately from TLS. Legacy checksum/no-integrity is never described as authenticated GCM.

## 0.2.2 — 2026-10-13

- Pin opencontrol-protocol =0.2.0 and submodule commit 35a5c848fb2a25a64da36268026906ce3757b974; keep wire semantics in the library.
- Complete single-message ROUTES replacement maintenance with flags 3 (split/explicit policy) and 11 (full), including empty snapshots.
- Repeat the initially advertised snapshot every 600 seconds in Active, including rekey, without reconnects, key resets or policy refresh.
- Report the linked protocol version through the library's VERSION constant in CLI/helper/status/logs.
- Preserve API v1, TLS, authentication, MFA handling, rekey algorithm, user policies and operator configuration.
- Add affected server tests, native interoperability acceptance and server quality CI; retain amd64/arm64 packages and SHA256SUMS.

## 0.2.1 — 2026-10-10

- Expose client-declared OS family, OS description, VPN-wire code and metadata availability in live sessions and the typed helper.
- Expose declared product version/build when present. Preserve the client's OS description with bounded UTF-8 sanitization.
- Keep missing legacy metadata nullable, unknown families explicit, authentication and the canonical VPN wire protocol unchanged.
- Preserve API version 1 and existing session fields; package upgrades retain operator users, certificates and configuration.

## 0.2.0 — 2026-10-10

- Add a root-only noninteractive JSON administration helper to amd64/arm64 packages.
- Add typed user/policy CRUD, complete registry apply, CAS revisions and explicit
  saved/applied outcomes without exposing passwords or PHC.
- Add safe configuration validation/application, confirmed reload, readiness,
  fixed-unit restart and verified rollback reporting.
- Add daemon instance identity and protection against stale session disconnects.
- Persist bounded close events with final byte/packet counters, unique event IDs,
  monotonic cursors, retention-gap reporting and explicit unclean-restart state.
- Default fresh configuration to full IPv4 routing and automatic outbound NAT
  interface selection; preserve existing operator configuration and external rules.
- Keep the canonical VPN protocol library and wire behavior unchanged.

OpenControl is independent and is not a Kerio/GFI product, affiliate, sponsored,
endorsed or supported solution. Interoperability statements apply to tested clients.

## 0.1.0

Initial IPv4 server, user/session CLI, split/full routing, journald, metrics and
Ubuntu amd64/arm64 packages.
