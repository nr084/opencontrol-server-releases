# OpenControl Releases

**[Русский](README_RU.md) · [English](README_EN.md)**

[OpenControl Server 0.9.1](https://github.com/nr084/opencontrol-server-releases/releases/tag/server-v0.9.1) · Ubuntu 26.04 · amd64 / arm64 · API v1

## Русский

Независимый IPv4 VPN-сервер, совместимый с проверенными клиентами Kerio Control VPN. Full/split routing, политики пользователей, TLS/UDP, NAT, CLI и JSON helper, статистика и метаданные ОС клиента.

**OpenControl не является продуктом Kerio / GFI и не связан с ними, не спонсируется, не одобрен и не поддерживается ими.**

- [Описание проекта](README_RU.md).
- [Руководство: установка, управление, обновление и удаление](SERVER_GUIDE_RU.md).
- [Административный API](ADMIN_API_RU.md).
- [Скачать пакеты и SHA256SUMS](https://github.com/nr084/opencontrol-server-releases/releases/tag/server-v0.9.1).

Версия 0.9.1 пересобрана на Rust 1.100.0 и выпущена как prerelease. Проверка с фирменным Linux-клиентом выполнена на 0.9.0. ARM64 тестируется под QEMU. Android и физический ARM-сервер пока не проверены. Установка не запускает службу автоматически.

## English

An independent IPv4 VPN server interoperating with tested Kerio Control VPN clients. Full/split routing, user policies, TLS/UDP, NAT, CLI and JSON helper, statistics and client OS metadata.

**OpenControl is not a Kerio / GFI product and is not affiliated with, sponsored by, endorsed by, or supported by Kerio / GFI.**

- [Project overview](README_EN.md).
- [Guide: installation, management, upgrades and removal](SERVER_GUIDE_EN.md).
- [Administration API](ADMIN_API.md).
- [Download packages and SHA256SUMS](https://github.com/nr084/opencontrol-server-releases/releases/tag/server-v0.9.1).

Version 0.9.1 is a prerelease rebuilt with Rust 1.100.0. The unmodified Linux client was tested with 0.9.0. ARM64 tests use QEMU. Android and native ARM hardware acceptance remain pending. Installation does not automatically start the service.
