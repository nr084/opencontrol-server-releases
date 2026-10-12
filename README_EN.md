# OpenControl binary releases

[Русский](README_RU.md) | **English**

Public binary releases of the independent OpenControl project: server and clients for supported platforms. Packages are attached to GitHub Releases, not committed to Git history.

**OpenControl is not a Kerio / GFI product and is not affiliated with, sponsored by, endorsed by, or supported by Kerio / GFI.** Names identify interoperability targets only; trademarks belong to their owners. No proprietary client binaries are distributed.

## OpenControl Server 0.2.1

IPv4 VPN server for Ubuntu 26.04 amd64/arm64. Full/split routing, DNS, user policies, NAT, root-only JSON administration with CAS, instance-aware sessions, sanitized OS metadata, durable close events, logs and metrics. The package includes `opencontrold`, `opencontrolctl` and `opencontrol-admin-helper`.

- [Download packages, SHA256SUMS and test results](https://github.com/nr084/opencontrol-server-releases/releases/tag/server-v0.2.1).
- [English administrator guide](SERVER_GUIDE_EN.md): install, configure, manage, monitor, upgrade and remove.
- [Local API v1](ADMIN_API.md).

Version 0.2.1 is a prerelease, tested with the unmodified Kerio Control VPN Client 9.6.1 (9306-p1) for Linux on amd64. ARM64 uses QEMU; native ARM hardware and Android acceptance remain pending. Other client versions are not implied compatible. Installation does not start the service or enable forwarding. Upgrades preserve operator users/configuration/certificates. Client metadata never authorizes access.

## Release conventions

Select eligible component/version tags such as `server-v0.2.1` from the release catalog; `/releases/latest` can omit prereleases. Server artifact names are `opencontrol-server_<version>_<arch>.deb`. Each release documents platform, limitations and checksums. Verify the selected package against SHA256SUMS before installation; integrity checksums are not an independent publisher signature.

[All releases](https://github.com/nr084/opencontrol-server-releases/releases) · [Server sources](https://github.com/nr084/opencontrol-server) · [Protocol library](https://github.com/nr084/opencontrol-protocol)
