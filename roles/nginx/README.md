# Роль `nginx`

Устанавливает Nginx и публикует каталог с PXE- или другим статическим содержимым через отдельный virtual host.

## Что делает роль

1. обновляет кэш APT и устанавливает Nginx;
2. сохраняет существующий site-файл через роль `backup`;
3. создаёт document root;
4. создаёт и включает `/etc/nginx/sites-available/pxe`;
5. отключает поставляемый пакетом сайт `default`;
6. проверяет полную конфигурацию командой `nginx -t`;
7. включает и запускает Nginx, а при изменениях выполняет reload.

## Требования

- Debian или Ubuntu с APT и systemd;
- запуск с `become: true`;
- доступ клиентов к выбранному TCP-порту.

## Переменные

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `nginx_packages` | `[nginx]` | Устанавливаемые пакеты |
| `nginx_pxe_root` | `/var/www/html` | Публикуемый каталог |
| `nginx_pxe_server_name` | `_` | Значение `server_name` |
| `nginx_pxe_listen_port` | `80` | TCP-порт |
| `nginx_backup_path` | `/opt/backups/nginx` | Каталог резервных копий |
| `nginx_backup_files` | site-файл `pxe` | Файлы для копирования |

## Пример

```yaml
- name: Configure HTTP content server
  hosts: nginx_servers
  become: true
  roles:
    - role: nginx
      vars:
        nginx_pxe_root: /var/www/html
        nginx_pxe_server_name: _
        nginx_pxe_listen_port: 80
```

Для каталога включён `autoindex`. Всё содержимое `nginx_pxe_root` считается доступным по HTTP.
