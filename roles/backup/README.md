# Роль `backup`

Создаёт на управляемом узле timestamp-копии указанных файлов и каталогов. Роль содержит общий механизм, ранее находившийся непосредственно в ролях `dhcp` и `nfs_server`.

Для каждого существующего источника создаётся путь вида:

```text
<backup_destination_path>/<YYYYMMDDTHHMMSS>_<имя источника>
```

Отсутствующие источники пропускаются без ошибки. Файлы и каталоги копируются модулем `ansible.builtin.copy` с `remote_src: true` и сохранением режима доступа. Архивы не создаются, старые копии автоматически не удаляются.

## Переменные

| Переменная | Значение по умолчанию | Назначение |
|---|---|---|
| `backup_destination_path` | `/opt/backups` | Каталог для резервных копий |
| `backup_source_files` | `[]` | Список файлов на управляемом узле |
| `backup_source_directories` | `[]` | Список каталогов на управляемом узле |

## Пример

```yaml
- name: Back up service configuration
  hosts: servers
  become: true
  roles:
    - role: backup
      vars:
        backup_destination_path: /opt/backups/example
        backup_source_files:
          - /etc/example.conf
        backup_source_directories:
          - /etc/example.d
```

Роль использует факт `ansible_date_time`, поэтому playbook должен собирать факты (`gather_facts: true`).
