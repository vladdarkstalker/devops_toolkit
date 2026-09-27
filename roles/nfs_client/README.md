# Роль `nfs_client`

Устанавливает клиентские NFS-утилиты, создаёт точку монтирования и подключает экспорт с сохранением записи в `/etc/fstab`.

## Что делает роль

1. обновляет кэш APT и устанавливает `nfs_packages`;
2. создаёт `nfs_mountpoint`;
3. монтирует `nfs_server_shared_address:nfs_server_shared_folder`;
4. делает подключение постоянным.

## Требования

- Debian или Ubuntu с APT;
- запуск с `become: true`;
- доступный NFS-сервер и экспорт;
- коллекция `ansible.posix`.

## Переменные

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `nfs_user`, `nfs_group` | `root` | Владелец и группа точки монтирования до подключения |
| `nfs_packages` | `[nfs-common]` | Клиентские пакеты |
| `nfs_mountpoint` | `/home/nfs` | Локальная точка монтирования |
| `nfs_server_shared_address` | обязательно | IP-адрес или DNS-имя сервера |
| `nfs_server_shared_folder` | `/home/nfs` | Экспортируемый путь |
| `nfs_mount_options` | `rw,sync,hard` | Строка опций монтирования |

Значения штатного сценария находятся в `playbooks/vars/nfs_client.yaml`.

## Пример

```yaml
- name: Configure NFS clients
  hosts: nfs_clients
  become: true
  roles:
    - role: nfs_client
      vars:
        nfs_server_shared_address: 192.168.10.10
        nfs_server_shared_folder: /home/nfs
        nfs_mountpoint: /mnt/nfs
        nfs_mount_options: rw,sync,hard
```

Изменение параметров обновляет соответствующую запись `/etc/fstab` и состояние монтирования.
