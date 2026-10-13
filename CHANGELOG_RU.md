# История изменений

**Русский** | [English](CHANGELOG.md)

## 0.2.2 — 2026-10-13

- Обновлена каноническая библиотека до точного `opencontrol-protocol =0.2.0`, submodule commit `35a5c848fb2a25a64da36268026906ce3757b974`. Wire semantics остаётся в библиотеке.
- Полные ROUTES-снимки завершают replacement maintenance: flags `3` для split/явной политики, `11` для full; пустые списки допустимы.
- В Active каждые 600 секунд повторяется первоначальный снимок, в том числе при rekey, без reconnect, сброса ключей и изменения политики объявления маршрутов.
- CLI/helper/status/logs сообщают фактически связанную версию протокола через `opencontrol_protocol::VERSION`.
- API v1, TLS, аутентификация/MFA handling, алгоритм rekey, политики пользователей и операторская конфигурация сохранены.
- Добавлены затронутые серверные тесты, native interoperability-приёмка и server quality CI. Формат пакетов amd64/arm64 и SHA256SUMS сохранён.

## 0.2.1

В активные сессии, CLI и typed helper добавлены очищенные клиентские метаданные ОС и версия/build. Для отсутствующих данных — null/unknown; неизвестные коды сохраняются без угадывания ОС. Метаданные не участвуют в авторизации.

## 0.2.0

Root-only JSON helper, CRUD/policies, CAS, saved/applied, подтверждённый reload, readiness, restart/rollback, instance ID и устойчивые события завершения с cursor и финальными счётчиками. Свежая конфигурация — full IPv4 с автоматическим outbound NAT interface.

## 0.1.0

Первый IPv4-сервер, CLI, split/full routing, journald, metrics и Ubuntu amd64/arm64 пакеты.

OpenControl — независимый проект, не продукт Kerio / GFI, не связан с ними, не спонсируется, не одобрен и не поддерживается ими. Совместимость ограничена проверенными клиентами.
