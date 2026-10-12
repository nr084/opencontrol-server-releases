# Локальный административный API v1 — OpenControl Server 0.2.1

**Русский** | [English](ADMIN_API.md)

`/usr/sbin/opencontrol-admin-helper` — отдельный root-only процесс без аргументов и listener. Он принимает один UTF-8 JSON через stdin и выдаёт один JSON через stdout. Максимум входа — 128 KiB; EOF нужен за 10 секунд. Веб-приложение и VPN daemon не должны работать от root. Пароли передаются только через stdin из памяти доверенного caller, никогда через argv, environment, файлы, историю shell или журналы.

Фиксированные пути: `/etc/opencontrol/opencontrol.toml`, `/etc/opencontrol/users.toml`, `/run/opencontrol/control.sock`. Конфигурационные файлы принадлежат root и не должны быть доступны группе/остальным для записи. Системный sandbox daemon сохраняется.

## Запрос и ответ

```json
{"api_version":1,"request_id":"example","operation":"users.list"}
```

Все запросы требуют `api_version:1`, `operation` и ASCII `request_id` длиной 1–128 из букв, цифр и `._:-`. Неизвестные операции и поля, включая вложенные, запрещены. Request ID — корреляция, не идемпотентность. Exit code — 0 при `ok:true`, иначе 1.

```json
{"api_version":1,"request_id":"example","ok":true,"saved":true,"applied":true,"revision":"sha256:...","applied_revision":"sha256:...","result":{}}
```

```json
{"api_version":1,"request_id":"example","ok":false,"saved":true,"applied":false,"revision":"sha256:...","applied_revision":"sha256:...","error":{"code":"RELOAD_FAILED","message":"Settings saved but not applied","retryable":true}}
```

Чтение возвращает `saved:null`, `applied:null`. Ошибка может сопровождаться сохранёнными настройками: всегда проверяйте оба флага. Ответы не содержат паролей, PHC, raw TOML или TLS-ключей. Ошибки обрабатывайте по `error.code`, а не по тексту: `INVALID_REQUEST`, `VALIDATION_FAILED`, `NOT_FOUND`, `ALREADY_EXISTS`, `REVISION_CONFLICT`, `RELOAD_FAILED`, `SERVICE_UNAVAILABLE`, `PERMISSION_DENIED`, `INTERNAL_ERROR`.

## Пользователи

| Операция | Дополнительные поля |
|---|---|
| `version` | — |
| `users.list` | — |
| `users.get` | `username` |
| `users.create` | `expected_revision`, `username`, `password`, `policy` |
| `users.update` | `expected_revision`, `username`, `patch` |
| `users.password` | `expected_revision`, `username`, `password` |
| `users.delete` | `expected_revision`, `username` |
| `users.apply` | `expected_revision`, `users` — полный реестр |

Политика: `enabled` (default true), `static_ip`, `max_sessions`, `idle_timeout`, `session_timeout`, `allowed_routes`, `description`. Остальные поля по умолчанию null. Таймауты — секунды, маршруты — IPv4 CIDR. Username неизменяемый; статический адрес допускает одну сессию. Используются существующие проверки пользователей, IP, маршрутов и Argon2.

Patch: отсутствует — оставить; null — наследование для optional fields; `enabled:null` сбрасывает к true. `allowed_routes:null` наследует серверную политику, `[]` ограничивает трафик исключениями сервера/DNS. Enforcement меняется сразу, объявленные клиенту маршруты — при переподключении. Disable/delete/password/static-IP changes закрывают старые сессии, снижение лимита оставляет самые старые разрешённые.

Ревизия покрывает точные байты глобальной конфигурации и полного users-файла, включая PHC. CAS и атомарные изменения используют общий `users.lock`. При конфликте перечитайте состояние; после timeout не повторяйте запись вслепую. Внешний root-редактор способен обойти блокировку — согласуйте доступ.

После сохранения helper подтверждает reload и проверяет readiness/фактически применённую ревизию. Недоступный daemon или pending restart оставляет `saved:true, applied:false`. `applied_revision` — принятая daemon ревизия, а не последняя отправленная.

`users.apply` принимает полных пользователей с валидным Argon2id `password_hash` только от доверенной root-автоматизации. Не предоставляйте generic pass-through браузеру. Чтение по-прежнему не выдаёт PHC. Не логируйте JSON запроса и не включайте его в аудит/исключения; авторизуйте каждую операцию в доверенном агенте.

## Конфигурация и служба

- `config.get`: безопасный полный типизированный объект.
- `config.validate`: поле `config`, проверка вместе с сохранёнными пользователями, без записи.
- `config.apply`: `config`, `expected_revision`, optional `restart:false`.

Разрешены `listen_address`, общий TCP/UDP `port`, `vpn` (сеть/интерфейс/адрес/пул/маршруты/DNS/NAT), `limits`, `logging_level`. Начинайте с `config.get`, не конструируйте неполный объект. TLS paths/profile, users/socket paths, публичность метрик и systemd definitions через API не меняются.

Пользователи и логирование reload live. Listener/network/routes/DNS/limits требуют restart и возвращают `result.restart_required:true`. При `restart:false` настройки остаются сохранёнными и pending; при true перезапускается только фиксированная `opencontrol.service`. Успех требует ожидаемой ревизии, новой instance, readiness и совпадения systemd MainPID. Отдельных операций start/stop/enable/disable нет.

Перед изменением сохраняется защищённый `/etc/opencontrol/opencontrol.toml.rollback`. Неудачный restart пытается CAS-защищённое восстановление и restart; `result.rollback_verified` показывает проверенный результат, `backup_retained` — наличие recovery data. Более новая конкурентная ревизия не перезаписывается; пользователи и сертификат/ключ не сбрасываются.

`readiness` возвращает daemon status; `sessions` — live sessions. `session.disconnect` требует `id` и исходный `instance_id` сессии. Не используйте повторно ID из старой instance. Disconnect асинхронный; разрешённый клиент может подключиться снова.

Status содержит `api_version`, `instance_id`, PID, `saved_revision`, `applied_revision`, `readiness`, безопасную running `configuration`, `previous_instance_id`, `previous_shutdown_unclean` и прежние поля. Сохранённая и работающая конфигурации могут различаться.

## ОС клиента

Дополнительные поля live sessions:

| Поле | Формат |
|---|---|
| `client_os_family` | windows / linux / macos / unknown |
| `client_os_name` | очищенная строка клиента либо null |
| `client_os_code` | исходный uint32 VPN-wire код либо null |
| `client_metadata_available` | bool — наличие блока |
| `client_version` | major.minor.patch либо null |
| `client_build` | uint16 либо null |

Wire-коды 1/2/3 — Windows/Linux/macOS; это не enum webadmin 0/1/2. Для wire v1/v2 или отсутствующих метаданных nullable-поля null, family unknown, availability false. Неизвестный код сохраняется с очищенным описанием, family остаётся unknown. Availability не означает доверие к сведениям.

Описание останавливается на первом NUL, декодируется с UTF-8 replacement, очищается от управляющих и bidi-formatting символов и ограничивается 128 UTF-8 байтами по границе символа. Пустой результат — null. Название ОС не заменяется семейным именем. Отображайте как экранированный plain text. Метаданные — самодекларация, не основание авторизации. Close events не изменены, API остаётся v1.

## Статистика и close events

Daemon хранит атомарный fsynced bounded journal `/var/lib/opencontrol/events.json`: 4096 close events, максимум 4 MiB. Startup проверяет возможность записи до ready. Не удаляйте/перезаписывайте outbox во время сбора.

`events.read` принимает `after` (uint64 cursor), `limit` (1–500). Доступен через helper и read-only daemon Unix API для root/членов socket group `opencontrol-admin`. Ответ: `events`, `cursor`, `oldest_cursor`, `latest_cursor`, `gap`, `healthy`. Gap означает потерю retention history или cursor за пределами этого outbox — не нулевой трафик.

Close record содержит event/instance/session IDs, username, creation/closure Unix seconds, duration, финальные rx/tx bytes/packets и reason. RX — клиент→сервер, TX — сервер→клиент; считаются внутренние IP-пакеты. Сохранение предшествует удалению сессии и освобождению IP. Ошибка фиксирует pending close, приостанавливает admission, делает readiness unhealthy и запускает retries; неудачное сохранение при shutdown даёт ошибку.

Committed events переживают restart/SIGKILL. Live counters находятся в RAM; unclean crash может потерять несохранённый хвост. `previous_shutdown_unclean` делает это явным. Не обещайте lossless billing. Финальные totals заменяют live snapshot, а не добавляются к ней.

Используйте один collector на узел: live sessions примерно раз в 5 секунд, metrics раз в 15 секунд. Дедупликация по node/event ID; запись событий и cursor — одной транзакцией. Метрики остаются loopback-only и сбрасываются с новой instance. Отличайте stale/unavailable от нулей. Journald служит диагностике, не заменяет cursor API.

## Маршрутизация

Свежий шаблон — full IPv4 и NAT, `outbound_interface="auto"` выбирает интерфейс реального IPv4-маршрута. Forwarding включает оператор; stop не выключает общий forwarding. Проверьте DNS и доверие сертификату. Пакетное обновление сохраняет operator conffiles.

Для внешней прозрачной маршрутизации отключите собственный NAT и отдельно обеспечьте routing/marks/DNS/fail-closed правила. OpenControl касается только своей nft table и не обходит пользовательские политики. Сервер IPv4-only; IPv6 bypass контролируется отдельно. Совместимость ограничена проверенными версиями/платформами.
