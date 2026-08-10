# Роль `nginx`

Роль создаёт HTTP-часть PXE-сервера. После получения kernel и initrd по TFTP установщик обращается к Nginx за содержимым дистрибутива или исходным ISO.

## Что делает роль

1. обновляет индекс APT;
2. устанавливает Nginx;
3. создаёт document root;
4. генерирует отдельный virtual host `pxe` с `autoindex`;
5. включает его символической ссылкой в `sites-enabled`;
6. удаляет поставляемый пакетом сайт `default`, чтобы исключить конфликт `default_server`;
7. проверяет итоговую конфигурацию командой `nginx -t`;
8. включает и запускает службу.

Роль не копирует установочные образы: это делает роль `iso`.

## Требования

- Debian/Ubuntu-подобная система с APT и systemd;
- права `become`;
- inventory-группа `[nginx_servers]`;
- доступ клиентов к заданному TCP-порту.

## Переменные

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `nginx_packages` | `[nginx]` | Пакеты HTTP-сервера |
| `nginx_pxe_root` | `/var/www/html` | Корень публикуемого дерева |
| `nginx_pxe_server_name` | `_` | Значение `server_name`; `_` принимает запросы с любым Host |
| `nginx_pxe_listen_port` | `80` | TCP-порт virtual host |

`nginx_pxe_root` должен совпадать с `iso_http_root`. Если меняется порт, URL в `roles/tftp/templates/default.j2` также должны содержать этот порт: одной переменной `pxe_http_server` для этого недостаточно.

## Inventory и запуск

```ini
[nginx_servers]
pxe01 ansible_host=192.168.0.24 ansible_user=user
```

```bash
ANSIBLE_CONFIG="$PWD/ansible.cfg" ansible-playbook \
  -i playbooks/inventory.ini playbooks/nginx.yaml \
  --limit pxe01 --ask-become-pass
```

## Управляемые объекты

| Объект | Назначение |
|---|---|
| `/var/www/html` | Document root по умолчанию |
| `/etc/nginx/sites-available/pxe` | Сгенерированный virtual host |
| `/etc/nginx/sites-enabled/pxe` | Ссылка, включающая сайт |
| `/etc/nginx/sites-enabled/default` | Удаляется ролью |
| `nginx` | Управляемая systemd-служба |

Для каталога включён `autoindex`, поэтому его содержимое доступно по HTTP без отдельной индексной страницы. Все файлы под document root следует считать публикуемыми.

## Связь с PXE

- роль `iso` заполняет `nginx_pxe_root` деревьями дистрибутивов;
- роль `tftp` формирует меню с адресом `pxe_http_server`;
- Nginx и ISO могут находиться на отдельном content-сервере, если URL меню указывают именно на него.
