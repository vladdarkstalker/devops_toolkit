# Роль `tftp`

Роль создаёт автономный TFTP-сервер и публикует загрузчики Syslinux для Legacy BIOS и UEFI x64. Она не зависит от роли `iso`: после отдельного запуска сервер уже отдаёт загрузчик и меню, хотя пункты установки станут рабочими только после публикации соответствующих kernel/initrd и HTTP-контента.

## Почему Syslinux собирается из исходников

Роль использует одну выбранную версию Syslinux для согласованного набора BIOS/UEFI-загрузчиков и `.c32`-модулей. Архив загружается с zytor.com, распаковывается в `/opt/syslinux` и компилируется на целевом сервере. Наличие собранного `bios/core/pxelinux.0` служит признаком завершённой сборки, поэтому повторный запуск не компилирует Syslinux заново.

## Порядок работы

1. обновляет индекс APT и устанавливает `tftpd-hpa`;
2. устанавливает компилятор, NASM, multilib и заголовки для Syslinux;
3. загружает и распаковывает архив Syslinux;
4. собирает BIOS и UEFI x64 firmware;
5. создаёт TFTP-корень и настраивает `/etc/default/tftpd-hpa`;
6. создаёт деревья `BIOS` и `EFIx64`;
7. копирует загрузчики, runtime-библиотеки и найденные `.c32`-модули;
8. генерирует одинаковое меню `pxelinux.cfg/default` для обоих режимов;
9. включает и запускает `tftpd-hpa`.

## Требования

- Debian/Ubuntu-подобный сервер с APT и systemd;
- права `become`;
- доступ сервера к URL архива Syslinux во время первой сборки;
- место в `/opt/syslinux` и во временном каталоге загрузки;
- inventory-группа `[tftp_servers]`;
- доступ клиентов к UDP/69 и динамическим UDP-портам TFTP-сеанса.

## Переменные службы и меню

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `tftp_packages` | `[tftpd-hpa]` | Runtime-пакеты |
| `tftp_root` | `/var/lib/tftpboot` | Публикуемый корень TFTP |
| `tftp_address` | `0.0.0.0:69` | Адрес и порт прослушивания |
| `tftp_options` | `--secure` | Параметры демона |
| `tftp_user` | `tftp` | Системный пользователь службы |
| `pxe_http_server` | `192.168.0.24` | Адрес HTTP-сервера в параметрах установщиков |
| `pxe_boot_menu_timeout` | `120` | Тайм-аут меню PXELINUX в десятых долях секунды |

## Переменные сборки Syslinux

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `tftp_syslinux_version` | `6.04-pre2` | Версия исходников |
| `tftp_syslinux_url` | URL zytor.com | Источник архива |
| `tftp_syslinux_download_dir` | `/tmp` | Временное размещение архива |
| `tftp_syslinux_install_root` | `/opt/syslinux` | Постоянный корень исходников и результатов сборки |
| `syslinux_build_dir` | вычисляется | Каталог распакованной версии |
| `tftp_syslinux_build_packages` | список пакетов | Зависимости компиляции |

Списки `tftp_subdirs`, `bios_specific_files` и `uefi_specific_files` описывают итоговую структуру и обязательные артефакты. Их обычно не переопределяют без изменения версии или архитектур Syslinux.

## Создаваемая структура

```text
/var/lib/tftpboot/
├── BIOS/
│   ├── pxelinux.0
│   ├── ldlinux.c32
│   ├── libutil.c32
│   ├── isolinux/*.c32
│   └── pxelinux.cfg/default
└── EFIx64/
    ├── syslinux.efi
    ├── ldlinux.e64
    ├── libcom32.c32
    ├── libutil.c32
    ├── isolinux/*.c32
    └── pxelinux.cfg/default
```

Роль `iso` позднее добавляет под `BIOS/<image>` и `EFIx64/<image>` kernel/initrd.

## Загрузочное меню

Меню находится в `roles/tftp/templates/default.j2`. Его пути должны одновременно соответствовать:

- `name` элемента `iso_images` — имени каталога в TFTP и HTTP;
- расположению kernel/initrd, опубликованных ролью `iso`;
- адресу `pxe_http_server`;
- параметрам командной строки конкретного установщика.

Добавление записи в `iso_images` само по себе не создаёт новый пункт меню: шаблон меню сейчас поддерживается явно.

### Особенности текущих пунктов

- Ubuntu Server и Ubuntu Desktop используют Casper с параметром `url=`. На этапе initramfs клиент загружает `installer.iso` целиком в оперативную память. Для тестовой VM объём RAM должен быть больше ISO с запасом для initramfs и установщика; практически рекомендуется не менее 6–8 GiB для текущего Server ISO и 12–16 GiB для Desktop ISO.
- Переход на установку без хранения live-образа в RAM потребует отдельного сетевого root-механизма, например NFS, которого текущий базовый стек пока не настраивает.

## Inventory и запуск

```ini
[tftp_servers]
pxe01 ansible_host=192.168.0.24 ansible_user=user
```

```bash
ANSIBLE_CONFIG="$PWD/ansible.cfg" ansible-playbook \
  -i playbooks/inventory.ini playbooks/tftp.yaml \
  --limit pxe01 --ask-become-pass
```

## Связь с другими ролями

- `dhcp_tftp_server` указывает на этот сервер;
- `iso_tftp_root` должен совпадать с `tftp_root`;
- `pxe_http_server` указывает на узел Nginx/ISO;
- поддерживаются BIOS и UEFI x64; UEFI IA32 в текущем дереве не публикуется.
