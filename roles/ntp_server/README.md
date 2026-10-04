# Роль `ntp_server`

Настраивает Chrony как NTP-сервер: синхронизирует узел с внешними источниками и разрешает указанным сетям получать от него время.

## Что делает роль

1. обновляет кэш APT и устанавливает Chrony;
2. сохраняет прежнюю конфигурацию через роль `backup`;
3. создаёт и проверяет candidate-конфигурацию;
4. применяет только успешно проверенный файл;
5. включает и запускает Chrony.

## Требования

- Debian или Ubuntu с APT и systemd;
- запуск с `become: true`;
- доступ к upstream-серверам по UDP/123;
- доступ клиентов к UDP/123 на сервере.

## Переменные

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `ntp_server_packages` | `[chrony]` | Устанавливаемые пакеты |
| `ntp_server_service_name` | `chrony` | systemd-служба |
| `ntp_server_config_path` | `/etc/chrony/chrony.conf` | Конфигурация Chrony |
| `ntp_server_upstreams` | три pool-источника | Внешние источники в формате `{address, options}` |
| `ntp_server_allowed_networks` | `[]` | Сети клиентов; пустой список не разрешает удалённым клиентам обращаться к серверу |
| `ntp_server_backup_path` | `/opt/backups/ntp_server` | Каталог резервных копий |
| `ntp_server_backup_files` | `[/etc/chrony/chrony.conf]` | Файлы для копирования |

Пример разрешённой сети:

```yaml
ntp_server_allowed_networks:
  - 192.168.0.0/24
```

Пример upstream-источников:

```yaml
ntp_server_upstreams:
  - address: 0.pool.ntp.org
    options: iburst
  - address: 1.pool.ntp.org
    options: iburst prefer
```

## Управляемые файлы

Роль создаёт доступный `chronyd` candidate-файл `<ntp_server_config_path>.ansible`, проверяет его и только после успешной проверки заменяет рабочую конфигурацию. Chrony перезапускается только при фактическом изменении рабочего файла.
