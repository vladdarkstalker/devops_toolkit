# Роль `iso`

Роль превращает локальные установочные ISO в контент, пригодный для PXE: переносит образ на управляемый сервер, временно монтирует его, публикует kernel/initrd по TFTP и установочное дерево по HTTP. Она не устанавливает TFTP или Nginx и поэтому в полном сценарии выполняется после этих ролей.

## Источник образов

ISO хранятся на Ansible controller внутри `roles/iso/files`. Поле `source` каждого элемента `iso_images` задаёт путь относительно этого каталога:

```text
roles/iso/files/
├── centos/CentOS-7-x86_64-DVD-1810.iso
├── ubuntu_server/ubuntu-26.04-live-server-amd64.iso
└── ubuntu_desktop/ubuntu-24.04.3-desktop-amd64.iso
```

Образы намеренно не должны храниться в Git. До запуска пользователь помещает их по ожидаемым путям либо переопределяет `iso_images`.

## Обработка одного образа

Для каждого элемента каталога роль:

1. проверяет marker успешной публикации в HTTP-каталоге;
2. решает, нужна ли обработка, учитывая `iso_force_republish`;
3. создаёт staging, mount, HTTP и TFTP-каталоги;
4. передаёт исходный ISO с controller в `iso_staging_root`;
5. проверяет, не смонтирован ли уже каталог назначения;
6. монтирует ISO read-only loop-устройством, если mount ещё не существовал;
7. копирует kernel и initrd в каталоги всех `iso_boot_modes`;
8. рекурсивно копирует содержимое ISO в HTTP-каталог;
9. при `publish_iso: true` дополнительно публикует цельный образ как `installer.iso`;
10. в секции `always` размонтирует только тот mount, который создала сама;
11. удаляет staging-копию ISO сразу после обработки;
12. создаёт `.pxe-published` только после успешной публикации и очистки.

Образы обрабатываются последовательно. Благодаря удалению staging-файла после каждого элемента в `/tmp` не накапливаются все ISO одновременно. Во время передачи Ansible дополнительно использует свой remote temporary directory, поэтому на целевом узле всё равно требуется свободное место для активного образа и временного файла передачи.

## Требования

- ISO-файлы на controller;
- права `become` для mount и системных каталогов;
- свободное место в staging, HTTP-корне и Ansible remote temp;
- утилиты `mountpoint`, `mount` и `umount` на целевом сервере;
- inventory-группа `[iso_servers]`;
- заранее настроенные TFTP и HTTP, если контент должен сразу обслуживаться клиентам.

## Основные переменные

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `iso_tftp_root` | `/var/lib/tftpboot` | Корень для kernel/initrd |
| `iso_http_root` | `/var/www/html` | Корень деревьев дистрибутивов |
| `iso_mount_root` | `/mnt/pxe` | Корень временных mount points |
| `iso_staging_root` | `/tmp/pxe-images` | Временная копия текущего ISO |
| `iso_force_republish` | `false` | Игнорировать marker и обработать каталог заново |
| `iso_boot_modes` | `[BIOS, EFIx64]` | Режимы, получающие копию kernel/initrd |
| `iso_images` | основной каталог ОС | Описание публикуемых образов |
| `iso_extra_images` | `[]` | Дополнительные локальные образы, не добавляемые в общие defaults |

`iso_tftp_root` должен совпадать с `tftp_root`, а `iso_http_root` — с `nginx_pxe_root`.

## Формат `iso_images`

```yaml
iso_images:
  - name: ubuntu_server
    title: Ubuntu Server 26.04
    source: ubuntu_server/ubuntu-26.04-live-server-amd64.iso
    kernel: casper/vmlinuz
    initrd: casper/initrd
    publish_iso: true
```

| Поле | Обязательное | Назначение |
|---|---|---|
| `name` | да | Безопасное имя каталогов TFTP, HTTP и mount point |
| `title` | да | Человекочитаемое имя в выводе Ansible |
| `source` | да | Относительный путь к ISO на controller |
| `kernel` | да | Путь к kernel внутри ISO |
| `initrd` | да | Путь к initrd внутри ISO |
| `publish_iso` | нет | Копировать ли исходный ISO в HTTP-каталог как `installer.iso` |

Имена файлов внутри разных выпусков дистрибутива меняются, поэтому `kernel` и `initrd` следует сверять с фактическим содержимым конкретного ISO.

## Итоговая структура

Для элемента с `name: ubuntu_server` создаются:

```text
/var/lib/tftpboot/BIOS/ubuntu_server/{vmlinuz,initrd}
/var/lib/tftpboot/EFIx64/ubuntu_server/{vmlinuz,initrd}
/var/www/html/ubuntu_server/<полное содержимое ISO>
/var/www/html/ubuntu_server/installer.iso       # publish_iso: true
/var/www/html/ubuntu_server/.pxe-published
```

Имена опубликованных kernel/initrd берутся как basename исходных путей. Шаблон TFTP-меню должен ссылаться на эти фактические имена.

## Marker и повторная публикация

Marker `/var/www/html/<name>/.pxe-published` означает, что прежний запуск дошёл до конца. При его наличии роль не передаёт ISO повторно. Marker не содержит checksum и не обнаруживает замену исходного файла автоматически.

Для осознанного обновления образов задайте:

```yaml
iso_force_republish: true
```

После успешного обновления верните значение в `false`, чтобы обычные повторные запуски снова были быстрыми.

## Добавление операционной системы

1. Поместите ISO в отдельный подкаталог `roles/iso/files`.
2. Добавьте полное описание в `iso_images`.
3. Укажите правильные пути к kernel/initrd внутри этого ISO.
4. Решите, нужен ли установщику цельный ISO по HTTP (`publish_iso`).
5. Добавьте соответствующий пункт в `roles/tftp/templates/default.j2`.
6. Проверьте, что параметры kernel используют URL с `pxe_http_server` и каталогом `name`.
7. При замене уже опубликованного образа увеличивать версию marker не нужно — используйте `iso_force_republish`.

## Inventory и запуск

```ini
[iso_servers]
pxe01 ansible_host=192.168.0.24 ansible_user=user
```

```bash
ANSIBLE_CONFIG="$PWD/ansible.cfg" ansible-playbook \
  -i playbooks/inventory.ini playbooks/iso.yaml \
  --limit pxe01 --ask-become-pass
```

Роль `iso` передаёт многогигабайтные файлы с controller. Запуск на `localhost` не эквивалентен публикации на PXE-сервере, если сервисы находятся на другом узле.

## Память PXE-клиента

Публикация по HTTP экономит место в TFTP, но не гарантирует потоковую установку. Текущие Ubuntu-пункты используют Casper `url=`, который скачивает `installer.iso` в RAM клиента: для Server рекомендуется 6–8 GiB, для Desktop — 12–16 GiB. Эти значения относятся к указанным в defaults версиям ISO и должны пересматриваться после их замены.
