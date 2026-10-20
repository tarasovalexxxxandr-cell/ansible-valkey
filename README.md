# Ansible Valkey

Primary и Replica Valkey, nftables и два экспортера метрик. Собственная роль
`valkey` управляет только Valkey; firewall и роли экспортеров подключены в playbook.
Названия задач — на английском, пояснения — на русском.

## Совместимость

Нужны apt, systemd, OpenSSH Server и пакеты `valkey-server`, `valkey-tools`,
`nftables` в подключённых репозиториях. Подходят совместимые Debian/Ubuntu;
перечисления допустимых релизов в роли нет. Недостающий пакет — ошибка установки,
проект не подключает неофициальные репозитории.

Ansible Core 2.16–2.21: нижняя граница учитывает `systemd_service` и требования
коллекций, верхняя — заявленную совместимость `prometheus.prometheus`.
Python контроллера и узлов должен поддерживаться выбранной версией Ansible
([матрица совместимости](https://docs.ansible.com/projects/ansible-core/2.20/reference_appendices/release_and_maintenance.html));
для Python 3.14 используйте Core 2.20 или новее. SSH по ключам и sudo готовятся
заранее. Конфигурация sshd через Ansible не изменяется.

Полный `molecule test` выполнен последовательно на двух окружениях:

| ОС | Valkey из apt | Ansible Core |
|---|---|---|
| Ubuntu 24.04 | 7.2.13 | 2.19.3 |
| Ubuntu 26.04 | 9.0.4 | 2.20.10 |

Molecule 25.9.0, Docker с cgroup v2, ARM64. Это проверенные примеры стенда,
а не ограничения роли. Оба прогона включали идемпотентность, репликацию,
восстановление drift, rollback firewall, SSH и повторный старт systemd.

## Подготовка

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install 'ansible-core>=2.16,<2.22' 'ansible-lint>=25.9,<27' 'molecule>=25.9,<26' 'yamllint>=1.35,<2' 'requests>=2.31,<3'
ansible-galaxy collection install -r requirements.yml
```

На macOS для `unarchive` нужен GNU tar: `brew install gnu-tar`.
На Linux контроллеру нужны tar/gzip. Роли экспортеров скачивают архивы с GitHub
на контроллер и проверяют SHA-256. При TLS/DNS-ошибках сначала проверьте доступ
контроллера к GitHub; отключать проверку TLS или checksum не нужно.

`requirements.yml` допускает исправления в совместимых ветках коллекций:

| Коллекция | Назначение |
|---|---|
| `prometheus.prometheus` | Роли `node_exporter` и `redis_exporter` |
| `community.general` | Зависимость ролей Prometheus; ограничение общей совместимой ветки |
| `community.docker` | Создание и очистка тестовых ресурсов, Docker connection plugin |

Транзитивные зависимости устанавливает Galaxy. Role объединяет задачи,
шаблоны и handlers; Collection распространяет роли, модули и плагины;
Module выполняет отдельную операцию. Использование двух ролей из коллекции —
полноценный пример Galaxy, отдельная внешняя роль не нужна.
Проверенные версии коллекций: Prometheus 0.30.2, General 11.4.x, Docker 4.8.x.

## Inventory и секрет

В `inventory/hosts.yml` адреса `192.0.2.11/12` — только примеры из документационной
подсети. Замените адреса, `ansible_user` и при необходимости `ansible_port`.
Группы задают топологию; `host_vars` — режимы и лимиты памяти.

По умолчанию Valkey слушает loopback и адрес основного IPv4-интерфейса из facts.
Для другой сети задайте `valkey_bind_address` и `valkey_bind` в Inventory.
Primary берётся из `valkey_primary`; адрес подключения по SSH может отличаться
от адреса репликации. `valkey_allowed_networks` задаёт доступ к порту Valkey.

Создайте Vault без открытого пароля на диске:

```bash
umask 077
openssl rand -hex 32 | sed 's/^/vault_valkey_password: /' | ansible-vault encrypt --output inventory/group_vars/valkey_nodes/vault.yml -
ansible-vault edit inventory/group_vars/valkey_nodes/vault.yml
```

Структура переменной: `vault_valkey_password: <64 hex-символа>`.
Vault-файл исключён из Git. Пароль Vault не передавайте через `-e` или командную
строку; используйте запрос либо локальный файл `0600` вне репозитория.
Valkey работает без TLS: используйте изолированную сеть.

## Применение и проверка

```bash
ansible-inventory --graph
ansible valkey_nodes -m ansible.builtin.ping --ask-vault-pass
ansible-playbook playbooks/site.yml --syntax-check --ask-vault-pass
ansible-playbook playbooks/site.yml --ask-vault-pass
ansible-playbook tests/verify.yml --ask-vault-pass
ansible-playbook playbooks/site.yml --ask-vault-pass
ansible-playbook playbooks/site.yml --check --diff --ask-vault-pass
```

`tests/verify.yml` не зависит от Docker: проверяет пакет, права, systemd, INFO,
режимы primary/replica, запрет запросов без правильного пароля, запись ключа на
Primary и чтение с Replica с ограниченным ожиданием. Ключ удаляется после теста.
В проверке Primary ожидается `changed=2` из-за записи и удаления ключа.
Повторный **site.yml** должен давать `changed=0`.

Проверяйте полную пару, без `--limit`. Для выборочного применения задавайте адреса
узлов явно в host_vars: facts невыбранного узла в этом запуске отсутствуют.

Check mode предназначен для уже установленного стенда: на чистом сервере ещё
нет пользователей, пакетов и каталогов. Секретные файлы скрыты из diff и логов.
Теги: `install`, `config`, `valkey`, `exporters`; `config` включает firewall.

## Firewall и метрики

Проект атомарно заменяет только таблицу `inet valkey` в
`/etc/nftables.d/valkey.nft`. Правила проходят `nft --check --file` до записи.
Чужой `/etc/nftables.conf` не перезаписывается. Входящие соединения закрыты,
кроме loopback, established/related, ICMP, DHCP, SSH и разрешённого доступа Valkey.
Для других сервисов задайте `firewall_allowed_tcp_ports`.

SSH-порты берутся из `sshd -T` и фактического `SSH_CONNECTION`, а не из числа 22.
По умолчанию SSH разрешён из любых сетей, как на подготовленном стенде;
аутентификация остаётся по ключам. Для ограничения источников задайте
`firewall_management_networks`; текущий источник SSH дополнительно сохраняется.
Дополнительные порты — `firewall_ssh_ports`.
Это также учитывает ProxyJump и различие внешнего NAT-порта и порта sshd.

`valkey-firewall.service` загружает только таблицу проекта и не очищает ruleset
при остановке. Valkey требует эту службу при загрузке. Изменение правил выполняется
через reload, без лишнего перезапуска Valkey. Перед применением запускается
независимый rollback-таймер: при потере новой SSH-сессии он восстанавливает старую
таблицу и её файл. Успешное подключение отменяет таймер. Время —
`firewall_rollback_timeout`, по умолчанию 90 секунд.

Reload и restart глобального `nftables.service` передаются службе проекта.
Сам глобальный сервис может очищать ruleset согласно своей конфигурации — чужие
приложения должны сохранять свои правила самостоятельно. Проект не изменяет эту
политику и рассчитан на выделенные учебные ВМ, а не произвольный общий firewall.

Экспортеры слушают `127.0.0.1:9100/metrics` и `127.0.0.1:9121/metrics`.
Upstream-роль `redis_exporter` поддерживает Valkey; namespace метрик — `valkey`.
Пароль передаётся через закрытый EnvironmentFile systemd, без пароля в argv.
Параметры и версии бинарников — в `group_vars/valkey_nodes/exporters.yml`.

```bash
curl -fsS http://127.0.0.1:9121/metrics | grep '^valkey_up '
systemctl status valkey-server valkey-firewall node_exporter redis_exporter
journalctl -u valkey-server -u valkey-firewall -n 30
```

Эти команды выполняются на ВМ. Ожидается `valkey_up 1`.
Для внешнего Prometheus нужны частный адрес прослушивания и правила firewall.

## Тестирование

```bash
yamllint .
ansible-lint
export ANSIBLE_CONFIG="$PWD/ansible.cfg"
cd roles/valkey
molecule test
molecule test
molecule destroy
```

Нужен Docker Engine с privileged-контейнерами, cgroup v2 и Docker CLI.
При нестандартном сокете настройте `DOCKER_HOST`. Контроллер работает через
Docker API; SSH проверяется между узлами на нестандартном порту.

Имена ресурсов получают случайный namespace, адреса выделяет Docker IPAM.
Можно задать `VALKEY_TEST_IMAGE` (например, `ubuntu:26.04`), `VALKEY_TEST_SUBNET`
и `VALKEY_TEST_SSH_PORT`. Для параллельных запусков задавайте
разные `MOLECULE_EPHEMERAL_DIRECTORY`; это разделяет состояние самого Molecule.
`destroy` удаляет только ресурсы своего namespace и допускает повторный запуск.

Последовательность: create → prepare → converge → check → idempotence → verify →
side_effect → converge → idempotence → verify → destroy. Side effect меняет порт
и память, проверяет PID handler, вносит drift и подтверждает, что общие проверки
его обнаруживают. Также проверяются существующая и новая SSH-сессии, чужая таблица,
перезапуски firewall, срабатывание rollback и повторный старт init контейнеров.
Перезапуск контейнера проверяет загрузку systemd, но не перезагрузку ядра ВМ.

На выделенных реальных ВМ перезагрузку проверяют отдельно:

```bash
ansible valkey_nodes -m ansible.builtin.reboot -a 'reboot_timeout=180' --ask-vault-pass
ansible-playbook tests/verify.yml --ask-vault-pass
```

## Короткий порядок демонстрации

1. Открыть `hosts.yml`, выполнить `ansible-inventory --graph`: группы и хосты.
2. Сравнить defaults, group_vars и host_vars: переопределение памяти и режима.
3. Открыть `site.yml`, задачи роли, Jinja2 и handler: связь template → notify.
4. Создать Vault и выполнить `site.yml`, затем `tests/verify.yml`: первый запуск.
5. Повторить `site.yml`: `changed=0`, объяснить идемпотентность.
6. Изменить `valkey_maxmemory` в host_vars; запустить `--check --diff`, затем site:
   меняется конфигурация и вызывается handler, секретный diff скрыт.
7. На ВМ изменить строку `maxmemory` в конфиге; запустить site: drift исправляется.
8. Показать `ansible-vault edit` и EnvironmentFile: секреты не передаются в argv.
9. Запустить `yamllint .`, `ansible-lint`, затем Molecule: статические и реальные
   функциональные проверки решают разные задачи.
