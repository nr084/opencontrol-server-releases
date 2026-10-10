# OpenControl Server 0.2.1: руководство администратора

IPv4 VPN-сервер для Ubuntu Server 26.04 LTS, amd64 и arm64. Реализует совместимость с проверенными клиентами Kerio Control VPN.

**OpenControl — независимый проект. Он не является продуктом Kerio / GFI, не связан с этими компаниями, не спонсируется, не одобрен и не поддерживается ими.** Названия используются только для обозначения совместимости; права на торговые марки принадлежат их владельцам. Фирменные клиентские бинарники не распространяются.

Версия 0.2.1 опубликована как prerelease. На Ubuntu amd64 проверен неизменённый Kerio Control VPN Client 9.6.1 (9306-p1) для Linux. ARM64 сборки и затронутые тесты проверены под QEMU; проверка на физическом ARM-сервере и Android пока не выполнена. Совместимость с другими версиями клиентов не гарантируется.

## Возможности

- Full tunnel для IPv4 либо split tunnel с заданными маршрутами; передача DNS-серверов и search domains.
- TLS для управления, зашифрованный UDP-трафик, защита от replay и обновление ключей через каноническую протокольную библиотеку.
- Локальные пользователи с Argon2id, включение/отключение, смена пароля, динамический или статический IPv4.
- Политики пользователя: разрешённые маршруты, число сессий, idle/session timeout, описание.
- NAT через собственную таблицу nftables, автоматический выбор исходящего интерфейса; возможность работы без собственного NAT с внешней маршрутизацией.
- CLI, локальный root-only JSON helper, CAS-ревизии и различие между сохранёнными и применёнными изменениями.
- Просмотр и отключение сессий, instance ID, устойчивые события завершения с финальными счётчиками и cursor.
- JSON-журналы в journald, loopback Prometheus metrics и диагностика.
- Отдельный непривилегированный пользователь daemon и ограниченная systemd-служба.

Сервер IPv4-only. IPv6 на клиенте и исключение обхода туннеля нужно учитывать отдельно. Метрики и административный socket не являются публичным HTTP API.

## Скачать и установить

Нужен Ubuntu Server 26.04 с systemd, `/dev/net/tun` и возможностью управлять сетью/nftables. В контейнере доступ к TUN и сетевые полномочия предоставляет оператор.

Скачайте `.deb` нужной архитектуры и `SHA256SUMS` из [релиза 0.2.1](https://github.com/nr084/opencontrol-server-releases/releases/tag/server-v0.2.1). Либо выполните:

```sh
sudo apt update
sudo apt install ca-certificates curl
arch=$(dpkg --print-architecture)
case "$arch" in amd64|arm64) ;; *) echo 'Unsupported architecture'; exit 1 ;; esac
base=https://github.com/nr084/opencontrol-server-releases/releases/download/server-v0.2.1
curl -fLO "$base/opencontrol-server_0.2.1_${arch}.deb"
curl -fLO "$base/SHA256SUMS"
grep "  opencontrol-server_0.2.1_${arch}.deb$" SHA256SUMS | sha256sum -c -
sudo apt install "./opencontrol-server_0.2.1_${arch}.deb"
```

Продолжайте только после успешной проверки SHA256. Команды ниже относятся к установленному серверу. Установка создаёт служебные учётные записи, конфигурацию и самоподписанный сертификат, если сертификата и ключа ещё нет. Служба автоматически не запускается; forwarding не включается.

## Первый запуск

Откройте конфигурацию:

```sh
sudoedit /etc/opencontrol/opencontrol.toml
```

Стандартные параметры нового пакета:

| Параметр | Значение |
|---|---|
| TCP и UDP | `0.0.0.0:4090` |
| TUN | `ocvpn0` |
| VPN-сеть | `10.253.0.0/24` |
| Адрес сервера | `10.253.0.1` |
| Пул клиентов | `10.253.0.10`–`10.253.0.250` |
| Маршрутизация | full IPv4 |
| DNS | `1.1.1.1` |
| NAT | включён, исходящий интерфейс `auto` |
| Метрики | `127.0.0.1:9108` |

Выберите VPN-подсеть, которая не пересекается с локальной сетью сервера и сетями клиентов. TCP и UDP должны использовать один порт. Разрешите этот порт в действующем firewall и, при необходимости, настройте проброс на маршрутизаторе. Не сбрасывайте существующий firewall: разрешение forwarded traffic зависит от его правил.

Для обычного доступа в Интернет оставьте full/NAT. `auto` выбирает интерфейс по реальному IPv4-маршруту. Для split tunnel измените соответствующий блок:

```toml
[vpn.routing]
mode = "split"
routes = ["192.168.50.0/24"]
```

Для доступа к внутренним сетям обеспечьте return routes либо осознанно выберите NAT. При `vpn.nat.enabled=false` маршрутизацию, DNS и fail-closed правила обеспечивает внешняя инфраструктура. OpenControl управляет только собственной таблицей `inet opencontrol`.

Включите forwarding, если он нужен этому узлу:

```sh
sysctl net.ipv4.ip_forward
sudo sysctl -w net.ipv4.ip_forward=1
```

Для сохранения после перезагрузки создайте операторский `/etc/sysctl.d/90-opencontrol.conf` со строкой `net.ipv4.ip_forward=1`. Запомните прежнее значение: forwarding может использоваться другими службами.

Проверьте конфигурацию, создайте пользователя и запустите сервер:

```sh
sudo opencontrolctl config check
sudo opencontrolctl user add alice
sudo systemctl enable --now opencontrol.service
sudo opencontrolctl status
```

Пароль вводится дважды без отображения. Не передавайте его аргументом командной строки.

## Подключение клиента и сертификат

В совместимом клиенте задайте публичный IP/DNS сервера, выбранный порт, имя пользователя и пароль. Для стандартного пакета порт — `4090`, нужны TCP и UDP.

Пакет создаёт локальный самоподписанный RSA3072 сертификат на 365 дней. Его fingerprint:

```sh
sudo openssl x509 -in /etc/opencontrol/server.crt -noout -fingerprint -sha256
sudo openssl x509 -in /etc/opencontrol/server.crt -noout -dates
```

Проверьте fingerprint через доверенный канал и подтвердите доверие в клиенте либо установите подходящий доверенный сертификат. Не отключайте проверку сертификата. Формат pin зависит от клиента. Автоматическое продление сертификата не предусмотрено.

Для собственного сертификата используйте пути из `[security]`, настройте владельца `root:opencontrol` и права `0640` для ключа и сертификата, выполните `config check` и restart. Не публикуйте private key.

После подключения проверьте сессии на сервере, доступ к нужным адресам и DNS на клиенте. В full mode дополнительно проверьте внешний IPv4 клиента.

## Пользователи через CLI

```sh
sudo opencontrolctl user list
sudo opencontrolctl user show alice
sudo opencontrolctl user add bob
sudo opencontrolctl user add fixed --static-ip 10.253.0.20
sudo opencontrolctl user passwd alice
sudo opencontrolctl user disable alice
sudo opencontrolctl user enable alice
sudo opencontrolctl user delete alice
```

Статический адрес должен пройти проверку конфигурации/уникальности; такой пользователь имеет одну одновременную сессию. CLI сохраняет изменения атомарно и при работающем daemon запрашивает reload. Если файл сохранён, но reload не прошёл, команда сообщает об этом. Изменения можно сохранять и при остановленной службе.

Отключение/удаление пользователя, смена пароля или статического адреса закрывают старые сессии. Для расширенных политик и автоматизации используйте JSON helper, описанный ниже. Не редактируйте реестр параллельно с административными операциями.

## Управление службой и сессиями

```sh
sudo systemctl start opencontrol.service
sudo systemctl stop opencontrol.service
sudo systemctl restart opencontrol.service
sudo systemctl enable opencontrol.service
sudo systemctl disable opencontrol.service
systemctl status opencontrol.service --no-pager
sudo opencontrolctl --json status
sudo opencontrolctl sessions
sudo opencontrolctl session show 42
sudo opencontrolctl session disconnect 42
sudo opencontrolctl diagnostics
```

`disable` отключает автозапуск, но не останавливает уже работающую службу; используйте `disable --now` для обоих действий. Отключение сессии асинхронное; разрешённый пользователь может подключиться снова. Для блокировки используйте `user disable`.

Пользователи и уровень логирования допускают reload:

```sh
sudo opencontrolctl reload
# либо
sudo systemctl reload opencontrol.service
```

Изменения listener, TLS, сети, DNS, маршрутов и лимитов требуют restart. После ручных изменений сначала выполняйте `sudo opencontrolctl config check`. Это CLI/systemd управление не заменяет CAS и проверенный rollback helper.

Read-only live CLI доступен root или членам `opencontrol-admin`; reload/disconnect требуют root. При необходимости предоставьте доверенному наблюдателю эту группу, затем обновите его login session:

```sh
sudo usermod -aG opencontrol-admin observer
```

## JSON helper для автоматизации

`/usr/sbin/opencontrol-admin-helper` запускается от root без аргументов, принимает один JSON через stdin и возвращает один JSON через stdout. Максимальный вход — 128 KiB; stdin должен закрыться за 10 секунд. Не запускайте веб-приложение или VPN daemon от root.

Безопасные запросы без секретов можно проверить из shell:

```sh
printf '%s' '{"api_version":1,"request_id":"version-1","operation":"version"}' | sudo -n /usr/sbin/opencontrol-admin-helper
printf '%s' '{"api_version":1,"request_id":"users-1","operation":"users.list"}' | sudo -n /usr/sbin/opencontrol-admin-helper
printf '%s' '{"api_version":1,"request_id":"ready-1","operation":"readiness"}' | sudo -n /usr/sbin/opencontrol-admin-helper
```

`sudo -n` требует уже разрешённого неинтерактивного root-доступа; оно не запрашивает пароль. Не выдавайте недоверенным пользователям доступ к helper: он разрешает менять весь реестр.

Операции: `version`, `users.list/get/create/update/password/delete/apply`, `config.get/validate/apply`, `readiness`, `sessions`, `session.disconnect`, `events.read`. Полный формат — в [ADMIN_API.md](ADMIN_API.md).

Алгоритм записи: прочитать `revision`, передать её как `expected_revision`, проверить `ok`, `saved`, `applied` и `applied_revision`. При `REVISION_CONFLICT` перечитать данные. `request_id` не обеспечивает идемпотентность: после timeout не повторять запись вслепую.

`users.create`/`users.password` получают plaintext password только через stdin из памяти доверенного caller. Не храните запрос в файле, argv, environment, shell history или журналах. Публичные ответы не содержат паролей/хешей.

Политика содержит `enabled`, `static_ip`, `max_sessions`, `idle_timeout`, `session_timeout`, `allowed_routes`, `description`. В patch отсутствующее поле не меняется, `null` сбрасывает к наследованию; `enabled:null` возвращает `true`. `allowed_routes:[]` ограничивает трафик исключениями сервера/DNS, `null` наследует серверную политику. Enforcement меняется сразу; объявленные клиенту маршруты обновляются при переподключении.

Для конфигурации получите полный объект `config.get`, измените его, вызовите `config.validate`, затем `config.apply` с CAS. `restart:false` оставляет restart-only изменения сохранёнными и pending; `restart:true` запускает фиксированный restart с проверкой readiness, новой instance, revision и MainPID. При ошибке проверяйте `rollback_verified` и `backup_retained`. Отдельных helper-операций start/stop/enable/disable нет — используйте фиксированную systemd-службу.

## Журналы и статистика

```sh
sudo journalctl -u opencontrol.service -f
sudo journalctl -u opencontrol.service --since '1 hour ago' --no-pager
curl -fsS http://127.0.0.1:9108/metrics
sudo opencontrolctl --json sessions
printf '%s' '{"api_version":1,"request_id":"events-1","operation":"events.read","after":0,"limit":100}' | sudo -n /usr/sbin/opencontrol-admin-helper
```

Live sessions включают instance ID, IP, состояние, возраст, idle time и RX/TX bytes/packets. Метрики дают active sessions, authentication successes/failures, трафик, protocol errors и rejected connections; counters сбрасываются при restart. Endpoint остаётся на loopback.

Close events содержат уникальный `event_id`, instance/session ID, пользователя, Unix timestamps, длительность, reason и финальные счётчики. RX — клиент→сервер, TX — сервер→клиент; учитываются внутренние IP-пакеты. Ответ `events.read` содержит `events`, `cursor`, `oldest_cursor`, `latest_cursor`, `gap`, `healthy`; limit — 1–500.

Для collector храните cursor транзакционно с событиями, дедуплицируйте по node/event ID, сессии идентифицируйте по node/instance/session ID. Финальные totals заменяют live snapshot, а не складываются с ней. Сохранённые события переживают restart, журнал ограничен 4096 событиями/4 MiB. `gap:true` означает неполную историю. Не удаляйте `/var/lib/opencontrol/events.json` при работающем сборе.

При дисковой ошибке новые подключения приостанавливаются и readiness становится unhealthy. После аварийного завершения `previous_shutdown_unclean` отмечает возможную потерю несохранённого хвоста активных сессий; точный биллинг такого хвоста не гарантируется.

## Обновление и резервная копия

Сохраняйте защищённую резервную копию `/etc/opencontrol` (там находятся PHC и private key). Для согласованной копии конфигурации, пользователей и outbox запланируйте остановку службы и сохраните также `/var/lib/opencontrol`. Доступ к копиям — только доверенным администраторам; не загружайте их в Git.

Скачайте новый пакет нужной архитектуры, проверьте его SHA256 и установите через `sudo apt install ./PACKAGE.deb`. Dpkg сохраняет операторские conffiles; при вопросе о конфигурации сравните изменения до выбора. Пакет не выполняет автоматический restart для обновления. Если служба работала, старый процесс может продолжать работу до явного restart.

```sh
sudo opencontrolctl config check
sudo systemctl restart opencontrol.service
sudo opencontrolctl --json status
```

Проверьте версию daemon, readiness, сессии и трафик. Старую резервную копию outbox не восстанавливайте поверх новой истории: это может вызвать откат cursor и потерю событий.

## Удаление

```sh
sudo systemctl disable --now opencontrol.service
sudo apt remove opencontrol-server
```

`remove` удаляет пакет и останавливает службу, оставляя conffiles. Для удаления пакетных conffiles:

```sh
sudo apt purge opencontrol-server
```

Даже purge не удаляет автоматически созданные сертификаты/ключи, `users.lock`, rollback-файл, state journal и служебные учётные записи. После нужной резервной копии и проверки, что эти каталоги относятся только к удаляемому OpenControl, их можно удалить явно:

```sh
sudo rm -rf -- /etc/opencontrol /var/lib/opencontrol
```

Служебного пользователя `opencontrol` и группы `opencontrol`/`opencontrol-admin` удаляйте отдельно только если они никем больше не используются. Не удаляйте клиентов Kerio или чужие сетевые правила вместе с сервером.

Удалите только добавленные вами firewall/port-forwarding правила и операторский sysctl-файл. Прежнее значение forwarding возвращайте лишь если другие приложения не нуждаются в нём. OpenControl при штатной остановке удаляет свои сетевые ресурсы и не выключает общий forwarding.

Скачанные `.deb` и `SHA256SUMS` можно удалить после установки/проверки, если локальный архив пакетов не нужен.

## Если подключение не работает

1. `systemctl status` и `journalctl`: ошибки запуска, занятый порт, сертификат, TUN, nftables.
2. `sudo opencontrolctl config check`: валидность конфигурации и пользователей.
3. TCP **и** UDP выбранного порта должны доходить до узла; проверьте firewall/NAT маршрутизатора.
4. `sessions` и `diagnostics`: авторизация, состояние сессии, VPN IP и сеть.
5. Если сессия есть, но трафика нет: forwarding, DNS, firewall FORWARD, NAT/return routes и пользовательские allowed routes.
6. Если settings сохранены, но не применены: сравните saved/applied revision, readiness и необходимость restart.

Подробнее: [DIAGNOSTICS.md](https://github.com/nr084/opencontrol-server/blob/v0.2.1/server/DIAGNOSTICS.md), [SECURITY.md](https://github.com/nr084/opencontrol-server/blob/v0.2.1/server/SECURITY.md), [CHANGELOG.md](https://github.com/nr084/opencontrol-server/blob/v0.2.1/server/CHANGELOG.md).

## ОС подключённого клиента (0.2.1)

`sessions`, `session show` и typed helper `sessions` возвращают дополнительные
`client_os_family`, `client_os_name`, `client_os_code`, `client_metadata_available`,
`client_version`, `client_build`. Например, штатный Linux-клиент передаёт
`linux`, `Ubuntu 26.04`, wire-код `2`, версию `9.6.1` и build `9306`.
Описание ОС остаётся клиентским, с ограничением длины и очисткой опасных символов;
некорректный UTF-8 заменяется безопасным replacement character. Wire v1/v2 без
метаданных дают unknown/null. Неизвестный код не превращается в известное семейство.
Это самодекларация клиента, не подтверждение ОС и не основание авторизации.
