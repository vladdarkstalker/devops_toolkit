# Роль `nexus`

Устанавливает Nexus Repository Manager из официального Unix-архива Sonatype, создаёт отдельного системного пользователя и управляет приложением через systemd.

## Что делает роль

1. устанавливает runtime-пакеты;
2. создаёт системные группу и пользователя;
3. скачивает официальный архив с проверкой SHA-256;
4. распаковывает версию в `/opt/sonatype` и создаёт стабильную ссылку `nexus`;
5. сохраняет существующую конфигурацию через роль `backup`;
6. настраивает `nexus.rc` и systemd unit;
7. включает и запускает Nexus.

## Требования

- Debian или Ubuntu с systemd;
- архитектура `x86_64` или `aarch64`;
- запуск с `become: true`;
- сетевой доступ к `download.sonatype.com`;
- роль `backup`.

## Переменные

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `nexus_version` | `3.96.4-01` | Версия официального архива |
| `nexus_download_url` | официальный Linux-архив | URL дистрибутива |
| `nexus_download_checksum` | официальный SHA-256 | Контрольная сумма |
| `nexus_user`, `nexus_group` | `nexus` | Runtime-учётная запись |
| `nexus_install_root` | `/opt/sonatype` | Корень установок |
| `nexus_install_link` | `/opt/sonatype/nexus` | Ссылка на активную версию |
| `nexus_service_name` | `nexus` | Имя systemd-службы |
| `nexus_backup_path` | `/opt/backups/nexus` | Каталог резервных копий |

## Запуск

```bash
ANSIBLE_CONFIG="$PWD/ansible.cfg" ansible-playbook \
  -i playbooks/inventory.ini playbooks/nexus.yaml --ask-become-pass
```

Роль не выполняет первоначальную настройку репозиториев, blob stores и пользователей через Nexus API. При обновлении задайте новую `nexus_version`; старая версия остаётся в `nexus_install_root`.

