# Роль `nfs_server`

Устанавливает `nfs-kernel-server`, создаёт экспортируемый каталог и добавляет управляемую запись в `/etc/exports`.

## Что делает роль

1. обновляет кэш APT и устанавливает серверные пакеты;
2. вызывает универсальную роль `backup`, чтобы сохранить существующий `/etc/exports` в каталог резервных копий;
3. создаёт экспортируемый каталог с заданными владельцем и группой;
4. добавляет запись вида `каталог сеть/префикс(опции)`;
5. включает и запускает `nfs-kernel-server`;
6. выполняет `exportfs -ra`, только если запись изменилась.

## Требования

- Debian или Ubuntu с APT и systemd;
- запуск с `become: true`;
- доступ клиентов к NFS-портам;
- согласованные UID/GID или выбранная политика squash.

## Переменные

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `nfs_user`, `nfs_group` | `root` | Владелец и группа каталога |
| `nfs_packages` | `[nfs-kernel-server]` | Серверные пакеты |
| `nfs_shared_folder` | `/home/nfs` | Экспортируемый каталог |
| `nfs_backup_path` | `/opt/backups/nfs` | Каталог резервных копий |
| `nfs_backup_files` | `[/etc/exports]` | Файлы для копирования |
| `nfs_shared_address`, `nfs_shared_subnet` | `192.168.0.0`, `24` | Сеть клиентов и длина префикса |
| `nfs_shared_access` | `rw` | `rw` или `ro` |
| `nfs_sync` | `sync` | `sync` или `async` |
| `nfs_secure` | `secure` | `secure` или `insecure` |
| `nfs_root_squash` | `root_squash` | `root_squash` или `no_root_squash` |
| `nfs_subtree_check` | `subtree_check` | `subtree_check` или `no_subtree_check` |

Топологические параметры штатного сценария находятся в `playbooks/vars/nfs_server.yaml`.

## Пример

```yaml
- name: Configure NFS server
  hosts: nfs_servers
  become: true
  roles:
    - role: nfs_server
      vars:
        nfs_shared_address: 192.168.10.0
        nfs_shared_subnet: 24
        nfs_shared_access: rw
        nfs_sync: sync
        nfs_secure: secure
        nfs_root_squash: root_squash
        nfs_subtree_check: no_subtree_check
```

Роль управляет только отмеченным блоком в `/etc/exports` и сохраняет остальные записи. Экспортируемый каталог создаётся с режимом `0777`; ограничьте его права и задайте общую группу, если свободная запись для всех пользователей не требуется.
