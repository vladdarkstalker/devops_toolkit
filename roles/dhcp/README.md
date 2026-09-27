# Роль `dhcp`

Устанавливает и настраивает `isc-dhcp-server` на Debian/Ubuntu. Роль управляет пулом IPv4-адресов, сетевыми параметрами клиентов и интерфейсами DHCP-службы. PXE-параметры и загрузочные файлы роль больше не настраивает.

## Что делает роль

1. обновляет кэш APT и устанавливает `dhcp_packages`;
2. вызывает универсальную роль `backup`, чтобы сохранить существующий `/etc/dhcp` в `dhcp_backup_path`;
3. создаёт candidate-файл `dhcpd.conf.ansible` и проверяет его командой `dhcpd -t`;
4. после успешной проверки обновляет рабочий `dhcpd.conf`;
5. настраивает интерфейсы в `/etc/default/isc-dhcp-server`;
6. включает и запускает `isc-dhcp-server`.

Изменение управляемого файла вызывает перезапуск службы. Ошибка проверки не заменяет рабочую конфигурацию.

## Требования

- Debian или Ubuntu с APT и systemd;
- запуск с `become: true`;
- статический адрес сервера;
- отсутствие другого несогласованного DHCP-сервера в том же широковещательном домене.

## Переменные

Параметры сети имеют тестовые значения по умолчанию и могут переопределяться в `playbooks/vars/dhcp.yaml`.

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `dhcp_user`, `dhcp_group` | `root` | Владелец и группа `dhcpd.conf` |
| `dhcp_packages` | `[isc-dhcp-server]` | Устанавливаемые пакеты |
| `dhcp_backup_path` | `/opt/backups/dhcp` | Каталог резервных копий |
| `dhcp_backup_dirs` | `[/etc/dhcp]` | Каталоги для копирования |
| `dhcp_subnet`, `dhcp_netmask` | `192.168.0.0`, `255.255.255.0` | Сеть и маска |
| `dhcp_range_start`, `dhcp_range_end` | `.100`, `.200` | Динамический пул |
| `dhcp_broadcast_address` | `192.168.0.255` | Broadcast-адрес |
| `dhcp_gateway` | `192.168.0.1` | Шлюз для клиентов |
| `dhcp_dns_servers` | Google и Cloudflare DNS | Список DNS-серверов |
| `dhcp_interfacesv4`, `dhcp_interfacesv6` | `[]` | Списки интерфейсов; пустой список означает отсутствие явного ограничения |

Все адреса подсети, диапазона, broadcast и шлюза должны относиться к одной сети.

## Пример

```yaml
- name: Configure DHCP server
  hosts: dhcp_servers
  become: true
  roles:
    - role: dhcp
      vars:
        dhcp_subnet: 192.168.10.0
        dhcp_netmask: 255.255.255.0
        dhcp_range_start: 192.168.10.100
        dhcp_range_end: 192.168.10.200
        dhcp_broadcast_address: 192.168.10.255
        dhcp_gateway: 192.168.10.1
        dhcp_dns_servers: [192.168.10.2]
        dhcp_interfacesv4: [ens19]
        dhcp_interfacesv6: []
```

Роль полностью управляет `/etc/dhcp/dhcpd.conf` и `/etc/default/isc-dhcp-server`; ручные изменения будут перезаписаны.
