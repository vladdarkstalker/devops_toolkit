# Роль `pxe`

Оркестрирует полный PXE-сервер для Legacy BIOS и UEFI x64: настраивает TFTP, HTTP и DHCP, публикует загрузчики Syslinux, ISO-образы, kernel/initrd и единое загрузочное меню.

## Что делает роль

1. устанавливает пакеты Syslinux;
2. вызывает роли `tftp` и `nginx`;
3. публикует BIOS/UEFI-загрузчики из установленных пакетов Syslinux;
4. копирует локальные ISO в HTTP-корень;
5. временно монтирует ISO и извлекает kernel/initrd в TFTP-корень;
6. создаёт меню для BIOS и UEFI;
7. вызывает роль `dhcp` и заменяет её конфигурацию проверенной PXE-конфигурацией.

DHCP включается последним, чтобы клиенты не получили ссылки на ещё не опубликованные файлы.

## Требования

- Debian или Ubuntu с APT и systemd;
- запуск с `become: true`;
- коллекция `ansible.posix`;
- роли `tftp`, `nginx`, `dhcp` и `backup`;
- локальные ISO, указанные переменными;
- доступ клиентов к DHCP, TFTP и HTTP.

## Переменные

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `pxe_user`, `pxe_group` | `root` | Владелец публикуемых файлов |
| `pxe_packages` | Syslinux packages | Пакеты загрузчиков |
| `pxe_tftp_server` | основной IPv4 узла | Адрес DHCP `next-server` |
| `pxe_http_server` | основной IPv4 узла | Адрес HTTP в параметрах загрузки |
| `pxe_boot_menu_timeout` | `120` | Тайм-аут меню в десятых долях секунды |
| `pxe_*_loader_source` | пути под `/usr/lib` | BIOS/UEFI-загрузчики из пакетов |
| `pxe_*_modules_source` | пути под `/usr/lib/syslinux` | Модули меню Syslinux |
| `pxe_boot_entries` | `[]` | ISO и параметры пунктов меню |

Каждая запись `pxe_boot_entries` содержит `name`, `distribution`, `version`, `label`, `menu_label`, `iso_source`, `kernel_source`, `initrd_source` и `append`.

## Запуск

```bash
ANSIBLE_CONFIG="$PWD/ansible.cfg" ansible-playbook \
  -i playbooks/inventory.ini playbooks/pxe.yaml --ask-become-pass
```

Штатный playbook задаёт локальные ISO в `playbooks/vars/pxe.yaml`. Каталог `roles/pxe/files` исключён из Git, поэтому ISO необходимо размещать на control node отдельно. Файлы в `files/autoinstall` сейчас пусты и роль их намеренно не публикует.

