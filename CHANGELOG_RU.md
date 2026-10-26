# История изменений

**Русский** | [English](CHANGELOG.md)

## 0.9.0 — 2026-10-26

- Точный pin библиотеки =0.9.0, commit cacf1a738e5013922ca9678d529d014fffc22d11. Включены feature legacy и обязательная runtime LegacyPolicy.
- Session epochs, generations, ACK-барьер, замена текущего слота peer и раздельная liveness делегированы ProtocolSession; завершение транспорта ограничено по времени.
- Сохраняемая security.allow_legacy выключена по умолчанию; доступна как limits.allow_legacy в API v1 и применяется reload. Отключение завершает сессии с любым сохранённым legacy-поколением ключей; AES-only продолжают работу.
- Запрещённый legacy отклоняется typed INVALID_REQUEST. Нет выдуманных локальных wire-кодов, автоматического downgrade и зависимости от TLS ciphersuite.
- ROUTES отправляются через advertisement_batches, включая START/END, пустые/многочастные/повторные снимки. Unchanged не подавляет повтор; safeLan не включает firewall.
- Проверки обычной роли сервера отделены от S2S initiator/acceptor; продуктовый S2S listener не включён.
- UDP cipher/integrity выводятся отдельно от TLS. Legacy checksum/no-integrity не приравниваются к GCM.

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
