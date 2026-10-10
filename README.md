# OpenControl binary releases

Binary releases of the independent OpenControl project: VPN server and clients for supported platforms. Installation packages are attached to GitHub Releases rather than committed to Git history.

## Compatibility and independence

OpenControl is designed for interoperability with compatible Kerio Control VPN clients. OpenControl Server 0.2.0 was tested with the unmodified Kerio Control VPN Client 9.6.1 (9306-p1) for Linux. Compatibility with other versions or products is not implied.

**OpenControl is not a Kerio or GFI product and is not affiliated with, sponsored by, endorsed by, or supported by Kerio or GFI.** Kerio and GFI names identify interoperability targets only; trademarks belong to their respective owners. Third-party client binaries are not redistributed here.

## Как пользоваться сервером

Текущая версия: **OpenControl Server 0.2.0** для Ubuntu 26.04, amd64/arm64.

- [Скачать 0.2.0: пакеты, SHA256SUMS и результаты проверки](https://github.com/nr084/opencontrol-server-releases/releases/tag/server-v0.2.0).
- [Полное руководство на русском](SERVER_GUIDE_RU.md): возможности, установка, сеть и сертификат, пользователи, служба, сессии, helper, статистика, обновление и удаление.
- [Контракт JSON helper API v1](ADMIN_API.md).

Пакет включает `opencontrold`, `opencontrolctl` и root-only `opencontrol-admin-helper`. Поддерживаются full/split IPv4, пользовательские политики, NAT, локальная автоматизация с CAS, журналы и устойчивые события завершённых сессий.

Установка не запускает службу и не включает forwarding. Версия — prerelease: Linux amd64 проверен с фирменным клиентом; arm64 проверен под QEMU, Android и физический ARM-сервер пока не проверены.

## Releases

Tags identify the component and version, for example `server-v0.1.0` or `android-v1.0.0`. Each release includes its target platform, installation instructions, limitations and SHA256 checksums.

- [Download releases](https://github.com/nr084/opencontrol-server-releases/releases)
- [Server sources](https://github.com/nr084/opencontrol-server)
- [Canonical protocol library](https://github.com/nr084/opencontrol-protocol)

For Ubuntu server packages, download the `.deb` and `SHA256SUMS` into the same directory and run `sha256sum --ignore-missing --strict -c SHA256SUMS` before installation. Checksums verify integrity; they are not a digital signature.
