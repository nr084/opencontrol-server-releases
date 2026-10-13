# Бинарные релизы OpenControl

**Русский** | [English](README_EN.md)

Публичный репозиторий бинарных релизов независимого проекта OpenControl: сервер и клиенты для поддерживаемых платформ. Пакеты прикрепляются к GitHub Releases, а не сохраняются в Git history.

**OpenControl не является продуктом Kerio / GFI, не связан с ними, не спонсируется, не одобрен и не поддерживается ими.** Названия обозначают только цели совместимости; права на торговые марки принадлежат владельцам. Фирменные клиентские бинарники не распространяются.

## OpenControl Server 0.2.2

IPv4 VPN-сервер для Ubuntu 26.04 amd64/arm64. Full/split routing, DNS, пользовательские политики, NAT, root-only JSON API с CAS, сессии с instance ID и очищенными метаданными ОС, устойчивые close events, журналы и метрики. В пакете: `opencontrold`, `opencontrolctl`, `opencontrol-admin-helper`.

- [Пакеты, SHA256SUMS и результаты тестов](https://github.com/nr084/opencontrol-server-releases/releases/tag/server-v0.2.2).
- [Русское руководство](SERVER_GUIDE_RU.md): установка, настройка, управление, мониторинг, обновление и удаление.
- [Локальный API v1](ADMIN_API_RU.md).

Версия — prerelease. На amd64 проверен неизменённый Kerio Control VPN Client 9.6.1 (9306-p1) для Linux. ARM64 проверен под QEMU; Android и физический ARM-сервер пока не проверены. Совместимость других версий клиентов не подразумевается. Установка не запускает службу и forwarding; обновление сохраняет пользователей, настройки и сертификаты. Метаданные клиента не участвуют в авторизации.

## Имена и выбор релиза

Выбирайте подходящий компонент/version tag, например `server-v0.2.2`, из каталога релизов с учётом prerelease; `/releases/latest` может их не показывать. Пакеты сервера: `opencontrol-server_<version>_<arch>.deb`. Проверяйте нужный пакет по SHA256SUMS до установки. Checksums подтверждают целостность, но не являются независимой подписью издателя.

[Все релизы](https://github.com/nr084/opencontrol-server-releases/releases) · [Исходники сервера](https://github.com/nr084/opencontrol-server) · [Протокольная библиотека](https://github.com/nr084/opencontrol-protocol)
