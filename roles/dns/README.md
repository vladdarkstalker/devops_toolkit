# Роль `dns`

Роль разворачивает внутренний BIND9 с авторитетными прямой и обратной зонами. Поддерживаются одиночный DNS-сервер и пара primary/secondary с передачей зон.

## Модель размещения

Роль определяет назначение серверов по порядку в inventory-группе, имя которой задаёт `dns_group_name`:

- первый узел — primary;
- второй узел — secondary, только если `dns_secondary_enabled: true`;
- остальные узлы ролью не используются как дополнительные secondary.

В одиночном режиме требуется один член группы. В режиме secondary роль до установки проверяет наличие как минимум двух узлов.

## Порядок работы

1. проверяет состав DNS-группы и вычисляет primary/secondary;
2. обновляет индекс APT и устанавливает BIND9 с утилитами проверки;
3. сохраняет существующие конфигурационные файлы в `/opt/backups/dns`;
4. задаёт режим запуска `named`: IPv4, IPv6 или dual-stack;
5. на всех DNS-узлах формирует ACL доверенных сетей, listeners, recursion и forwarders;
6. на primary создаёт объявления зон и их базы с SOA, NS, A и PTR;
7. валидирует конфигурацию через `named-checkconf`, зоны — через `named-checkzone`;
8. на secondary объявляет slave-зоны и адрес primary;
9. применяет handlers после успешной проверки и включает BIND в автозагрузку.

## Требования

- Debian/Ubuntu-подобные серверы с APT и systemd;
- права `become`;
- статические адреса и inventory-группа `[dns_servers]`;
- сетевой доступ клиентов к TCP/UDP 53;
- для secondary — TCP/53 между secondary и primary для zone transfer.

## Одиночный DNS на PXE-сервере

```ini
[dns_servers]
pxe01 ansible_host=192.168.0.24 ansible_user=user
```

```yaml
dns_secondary_enabled: false
```

В этом режиме зона содержит одну NS-запись, а transfer зоны запрещён.

## Primary и secondary

```ini
[dns_servers]
dns01 ansible_host=192.168.0.2 ansible_user=user
dns02 ansible_host=192.168.0.3 ansible_user=user
```

```yaml
dns_secondary_enabled: true
```

При совмещённом PXE-сервере `pxe01` может оставаться первым узлом, а отдельный `dns02` — вторым. В `playbooks/vars/pxe.yaml` список DHCP DNS тогда обычно выглядит так:

```yaml
dhcp_dns_servers:
  - 192.168.0.24
  - 192.168.0.25
```

## Переменные топологии и зон

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `dns_group_name` | `dns_servers` | Inventory-группа, порядок которой задаёт роли узлов |
| `dns_secondary_enabled` | `false` | Включает настройку второго DNS и zone transfer |
| `domain` | `dark.side.com` | Имя прямой зоны |
| `dns_trusted_networks` | `[192.168.0.0/24]` | ACL сетей, которым разрешены запросы и рекурсия |
| `dns_reverse_zone` | `0.168.192.in-addr.arpa` | Имя обратной зоны |
| `dns_reverse_network` | `192.168.0` | Первые три октета адресов текущей `/24` зоны |
| `dns_zone_records` | `pxe -> 192.168.0.24` | Дополнительные A/PTR-записи |
| `dns_zone_serial` | `1` | Серийный номер SOA |
| `dns_forwarders` | Google DNS | Рекурсивные upstream-резолверы |
| `dualstack` | `ipv4` | `ipv4`, `ipv6` или `dual` |

Параметры таймеров SOA: `dns_zone_refresh`, `dns_zone_retry`, `dns_zone_expire`, `dns_zone_negative_ttl`. Пути управляемой конфигурации задаются `optionsfile` и `localfile`, резервных копий — `dns_backup_path` и `dns_backup_files`.

### Ограничение обратной зоны

Текущие шаблоны рассчитаны на `/24`: `dns_reverse_network` содержит три первых октета, а PTR строится из последнего октета каждого `address`. Для другой длины префикса недостаточно изменить только имя зоны — потребуется адаптировать шаблон обратной зоны.

### Добавление записей

```yaml
dns_zone_records:
  - name: pxe
    address: 192.168.0.24
  - name: repo
    address: 192.168.0.25
dns_zone_serial: 2
```

Имена задаются без суффикса домена. При каждом изменении зоны увеличивайте `dns_zone_serial`, иначе secondary может не загрузить новую версию.

## Управляемые файлы

| Путь | Узлы | Содержимое |
|---|---|---|
| `/etc/default/named` | все | Аргументы семейства адресов |
| `/etc/bind/named.conf.options` | все | ACL, recursion, forwarders и listeners |
| `/etc/bind/named.conf.local` | все | Объявления primary либо secondary зон |
| `/etc/bind/zones/db.<domain>` | primary | Прямая зона |
| `/etc/bind/zones/db.<reverse-zone>` | primary | Обратная зона |
| `/var/cache/bind/` | secondary | Полученные от primary slave-зоны, управляемые BIND |
| `/opt/backups/dns/` | все | Копии конфигурации до замены |

Пакет BIND создаёт базовую структуру `/etc/bind`, но содержание `named.conf.local` для зон проекта создаёт и далее полностью контролирует роль.

## Запуск

```bash
ANSIBLE_CONFIG="$PWD/ansible.cfg" ansible-playbook \
  -i playbooks/inventory.ini playbooks/dns.yaml \
  --limit pxe01 --ask-become-pass
```

При включённом secondary limit должен охватывать оба DNS-узла, иначе конфигурация пары будет применена не полностью.

Конфигурационная схема основана на статье [DigitalOcean: How To Configure BIND as a Private Network DNS Server on Ubuntu 22.04](https://www.digitalocean.com/community/tutorials/how-to-configure-bind-as-a-private-network-dns-server-on-ubuntu-22-04), но inventory, шаблоны и режим одиночного сервера реализованы внутри проекта.
