# Роль `ntp_client`

Настраивает Chrony как NTP-клиент. Роль устанавливает пакет, сохраняет прежнюю конфигурацию, записывает серверы времени и запускает службу.

## Что делает роль

1. обновляет кэш APT и устанавливает Chrony;
2. сохраняет прежнюю конфигурацию через роль `backup`;
3. создаёт и проверяет candidate-конфигурацию;
4. применяет только успешно проверенный файл;
5. включает и запускает Chrony.

## Требования

- Debian или Ubuntu с APT и systemd;
- запуск с `become: true`;
- доступ к выбранным серверам времени по UDP/123.

## Переменные

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `ntp_client_packages` | `[chrony]` | Устанавливаемые пакеты |
| `ntp_client_service_name` | `chrony` | systemd-служба |
| `ntp_client_config_path` | `/etc/chrony/chrony.conf` | Конфигурация Chrony |
| `ntp_client_servers` | два источника | Серверы времени в формате `{address, options}` |
| `ntp_client_backup_path` | `/opt/backups/ntp_client` | Каталог резервных копий |
| `ntp_client_backup_files` | `[/etc/chrony/chrony.conf]` | Файлы для копирования |

Пример использования внутреннего сервера:

```yaml
ntp_client_servers:
  - address: 192.168.0.24
    options: iburst
```

## Управляемые файлы

Роль создаёт доступный `chronyd` candidate-файл `<ntp_client_config_path>.ansible`, проверяет его и только после успешной проверки заменяет рабочую конфигурацию. Chrony перезапускается только при фактическом изменении рабочего файла.
