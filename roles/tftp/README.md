# Роль `tftp`

Устанавливает и настраивает автономный сервер `tftpd-hpa` на Debian/Ubuntu. Роль публикует указанный каталог по TFTP, но не создаёт PXE-загрузчики, меню или образы.

## Что делает роль

1. обновляет кэш APT и устанавливает пакеты;
2. перед изменением сохраняет `/etc/default/tftpd-hpa` через роль `backup`;
3. создаёт TFTP-корень;
4. генерирует `/etc/default/tftpd-hpa`;
5. включает и запускает службу.

## Требования

- Debian или Ubuntu с APT и systemd;
- запуск с `become: true`;
- доступ клиентов к UDP/69 и динамическим UDP-портам TFTP.

## Переменные

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `tftp_packages` | `[tftpd-hpa]` | Устанавливаемые пакеты |
| `tftp_root` | `/var/lib/tftpboot` | Публикуемый каталог |
| `tftp_address` | `0.0.0.0:69` | Адрес и UDP-порт |
| `tftp_options` | `--secure` | Параметры демона |
| `tftp_user` | `tftp` | Владелец TFTP-корня и пользователь службы |
| `tftp_backup_path` | `/opt/backups/tftp` | Каталог резервных копий |
| `tftp_backup_files` | `[/etc/default/tftpd-hpa]` | Копируемые до изменения файлы |

## Пример

```yaml
- name: Configure TFTP
  hosts: tftp_servers
  become: true
  roles:
    - role: tftp
      vars:
        tftp_root: /var/lib/tftpboot
        tftp_address: 0.0.0.0:69
        tftp_options: --secure
```

Для PXE загрузчики и конфигурацию меню необходимо положить в `tftp_root` отдельной ролью или задачами.
