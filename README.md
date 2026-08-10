# DevOps Toolkit: автоматизация PXE-инфраструктуры

`devops_toolkit` — набор Ansible-ролей и playbooks для развёртывания сервера
сетевой установки операционных систем по PXE в Debian/Ubuntu-среде.

Проект автоматизирует полный путь PXE-клиента: получение сетевых параметров по
DHCP, загрузку BIOS/UEFI-загрузчика и kernel/initrd по TFTP, доступ к
установочному ISO или репозиторию по HTTP и разрешение внутренних имён через DNS.

Текущая конфигурация рассчитана на лабораторную сеть `192.168.0.0/24`, в которой
единый сервер `pxe01` имеет адрес `192.168.0.24`. Все значения можно изменить в
одном файле — [`playbooks/vars/pxe.yaml`](playbooks/vars/pxe.yaml).

> Проект находится в активной разработке. Перед использованием в production
> необходимо адаптировать сетевую схему, inventory, firewall, политику хранения
> образов и секретов. Текущий статус и незавершённые компоненты описаны в
> [`docs/ROADMAP.md`](docs/ROADMAP.md).

## Содержание

- [Назначение проекта](#назначение-проекта)
- [Как работает PXE-цепочка](#как-работает-pxe-цепочка)
- [Состав проекта](#состав-проекта)
- [Поддерживаемые сценарии](#поддерживаемые-сценарии)
- [Требования](#требования)
- [Подготовка управляющего узла](#подготовка-управляющего-узла)
- [Подготовка inventory](#подготовка-inventory)
- [Единый файл сетевых переменных](#единый-файл-сетевых-переменных)
- [Подготовка ISO](#подготовка-iso)
- [Проверка перед запуском](#проверка-перед-запуском)
- [Запуск полного PXE-стека](#запуск-полного-pxe-стека)
- [Запуск отдельных компонентов](#запуск-отдельных-компонентов)
- [Порядок выполнения полного playbook](#порядок-выполнения-полного-playbook)
- [Добавление новой операционной системы](#добавление-новой-операционной-системы)
- [Создаваемые сервисы и файлы](#создаваемые-сервисы-и-файлы)
- [Переменные и их приоритет](#переменные-и-их-приоритет)
- [Сетевая и эксплуатационная модель](#сетевая-и-эксплуатационная-модель)
- [Документация ролей](#документация-ролей)

## Назначение проекта

Проект решает следующие задачи:

- автоматически устанавливает и конфигурирует DHCP-сервер для PXE-клиентов;
- разворачивает одиночный или primary/secondary внутренний BIND9;
- устанавливает Nginx как HTTP-хранилище установочных материалов;
- устанавливает TFTP и собирает Syslinux для BIOS и UEFI x64;
- создаёт единое PXELINUX-меню выбора операционной системы;
- копирует ISO на управляемый сервер, монтирует их read-only и публикует
  kernel/initrd, ISO и содержимое установочного репозитория;
- позволяет развернуть каждый сервис отдельно или выполнить полный сценарий
  одним составным playbook;
- предоставляет отдельную роль QEMU/KVM для подготовки виртуализационного хоста,
  на котором можно создавать тестовые PXE-клиенты.

Проект не управляет сетевыми интерфейсами хоста, VLAN, маршрутизацией, firewall,
коммутаторами, DHCP relay, UEFI Secure Boot, TLS для HTTP и жизненным циклом
секретов. Эти параметры должны быть подготовлены отдельно.

## Как работает PXE-цепочка

```text
PXE-клиент
    |
    | 1. DHCP Discover
    v
isc-dhcp-server
    |  выдаёт IP, gateway, DNS, next-server и имя boot-файла
    |
    | 2. TFTP GET BIOS/pxelinux.0 или EFIx64/syslinux.efi
    v
tftpd-hpa + Syslinux
    |  читает BIOS/pxelinux.cfg/default или EFIx64/pxelinux.cfg/default
    |
    | 3. загружает kernel и initrd выбранной ОС
    v
Linux installer
    |
    | 4. получает installer.iso или HTTP-репозиторий
    v
Nginx /var/www/html

BIND9 обслуживает внутреннюю forward/reverse DNS-зону на всех этапах.
```

DHCP выбирает boot-файл по option 93:

| Архитектура клиента | DHCP-код | Выдаваемый файл |
|---|---:|---|
| Legacy BIOS | `00:00` | `BIOS/pxelinux.0` |
| UEFI x64 | `00:07`, `00:09` | `EFIx64/syslinux.efi` |
| UEFI IA32 | `00:06` | `EFIia32/syslinux.efi` |

Роль TFTP в текущем состоянии публикует BIOS и UEFI x64. Ветка EFI IA32 в DHCP
зарезервирована, но соответствующее дерево TFTP ещё не реализовано.

## Состав проекта

```text
.
├── ansible.cfg
├── README.md
├── docs/
│   └── ROADMAP.md
├── playbooks/
│   ├── inventory.example.ini
│   ├── vars/
│   │   └── pxe.yaml
│   ├── pxe.yaml
│   ├── dhcp.yaml
│   ├── dns.yaml
│   ├── nginx.yaml
│   ├── tftp.yaml
│   ├── iso.yaml
│   └── qemu_kvm.yaml
└── roles/
    ├── dhcp/
    ├── dns/
    ├── nginx/
    ├── tftp/
    ├── iso/
    ├── qemu_kvm/
    ├── nfs/
    └── ntp/
```

### Реализованные PXE-роли

| Роль | Назначение | Inventory-группа | Автономный playbook |
|---|---|---|---|
| `dhcp` | DHCP и выбор BIOS/UEFI boot-файла | `dhcp_servers` | `playbooks/dhcp.yaml` |
| `dns` | BIND9, forward/reverse зоны | `dns_servers` | `playbooks/dns.yaml` |
| `nginx` | HTTP publication root | `nginx_servers` | `playbooks/nginx.yaml` |
| `tftp` | TFTP, сборка Syslinux, boot menu | `tftp_servers` | `playbooks/tftp.yaml` |
| `iso` | Публикация ISO, kernel и initrd | `iso_servers` | `playbooks/iso.yaml` |

### Дополнительная реализованная роль

| Роль | Назначение | Inventory-группа | Playbook |
|---|---|---|---|
| `qemu_kvm` | QEMU/KVM, libvirt и OVMF | `qemu_kvm_servers` | `playbooks/qemu_kvm.yaml` |

### Роли в разработке

`nfs` и `ntp` пока содержат только заготовки defaults и не участвуют в PXE
playbooks. Подробности вынесены в [`docs/ROADMAP.md`](docs/ROADMAP.md), чтобы
руководство по рабочим компонентам не смешивалось с планами разработки.

## Поддерживаемые сценарии

### Один PXE-сервер

Сценарий по умолчанию размещает DNS, Nginx, TFTP, ISO publication и DHCP на
сервере `pxe01` с адресом `192.168.0.24`.

```text
pxe01 (192.168.0.24)
├── bind9
├── nginx
├── tftpd-hpa
├── Syslinux files
├── ISO publication
└── isc-dhcp-server
```

Все inventory-группы содержат одно и то же имя `pxe01`. Переменные подключения
достаточно объявить один раз — Ansible объединит членство хоста в группах.

### Распределённые сервисы

DHCP, DNS, Nginx и TFTP можно назначить разным хостам через inventory. При этом:

- `dhcp_tftp_server` должен указывать на адрес TFTP-сервера;
- `pxe_http_server` должен указывать на адрес Nginx;
- `dhcp_dns_servers` должен содержать адреса реальных DNS-серверов;
- роль `iso` должна выполняться на сервере, где локально доступны и
  `iso_tftp_root`, и `iso_http_root`;
- если Nginx и TFTP находятся на разных серверах, текущую роль `iso` необходимо
  адаптировать для раздельной публикации или использовать общую файловую систему.

### DNS primary/secondary

По умолчанию используется один DNS на `pxe01`:

```yaml
dns_secondary_enabled: false
```

Для двух DNS-серверов включите secondary и укажите два хоста в `dns_servers`.
Первый хост становится primary, второй — secondary. Полный пример приведён в
[`roles/dns/README.md`](roles/dns/README.md).

## Требования

### Управляющий узел Ansible

- Linux или WSL;
- Ansible Core 2.16 или совместимая версия;
- Python 3;
- SSH-доступ к управляемым узлам;
- свободное место для локальных ISO в `roles/iso/files`;
- сетевой доступ к target-хостам и, для сборки Syslinux, к `zytor.com`.

Проект проверялся с Ansible Core `2.16.3` и Python `3.12`.

### Управляемые серверы

- Debian/Ubuntu с `systemd` и APT;
- Python для выполнения Ansible-модулей;
- пользователь с правом `become`/`sudo`;
- доступ к пакетным репозиториям;
- достаточно RAM/CPU для компиляции Syslinux;
- достаточно дискового пространства для ISO и распакованных HTTP-репозиториев;
- PXE-сервер и клиенты должны находиться в одном L2 broadcast domain либо сеть
  должна предоставлять правильно настроенный DHCP relay;
- на клиентском сегменте не должно быть конкурирующего DHCP-сервера, если он не
  согласован с этой PXE-схемой.

### Требуемые сетевые порты

Проект не открывает firewall автоматически.

| Сервис | Протокол/порт | Направление |
|---|---|---|
| DHCP server | UDP `67` | клиенты → PXE/DHCP |
| DHCP client | UDP `68` | PXE/DHCP → клиенты |
| TFTP | UDP `69` и динамический TFTP traffic | клиенты → TFTP |
| DNS | UDP/TCP `53` | клиенты → DNS |
| HTTP | TCP `80` | клиенты → Nginx |
| SSH | TCP `22` | Ansible controller → серверы |

## Подготовка управляющего узла

Клонируйте проект и перейдите в его корень:

```bash
git clone <repository-url> devops_toolkit
cd devops_toolkit
```

Проверьте Ansible:

```bash
ansible-playbook --version
```

Проект содержит `ansible.cfg`:

```ini
[defaults]
ask_pass = False
gathering = explicit
roles_path = roles
```

Playbooks дополнительно используют относительные пути `../roles/<role>`, поэтому
поиск ролей не зависит от `roles_path`. Если проект расположен на WSL-mounted
Windows-диске, конфигурацию можно выбрать явно для текущей shell-сессии:

```bash
export ANSIBLE_CONFIG="$PWD/ansible.cfg"
```

Рекомендуется использовать SSH-ключ:

```bash
ssh-keygen -t ed25519
ssh-copy-id <remote-user>@192.168.0.24
```

Не запускайте `ssh-copy-id` через `sudo`: Ansible запускается от текущего
пользователя controller и должен иметь доступ к его приватному ключу.

## Подготовка inventory

Создайте рабочий inventory из шаблона:

```bash
cp playbooks/inventory.example.ini playbooks/inventory.ini
```

`playbooks/inventory.ini` добавлен в `.gitignore`, поскольку может содержать
локальные адреса и параметры доступа.

### Один сервер `pxe01`

```ini
[dns_servers]
pxe01 ansible_host=192.168.0.24 ansible_user=user

[nginx_servers]
pxe01

[tftp_servers]
pxe01

[iso_servers]
pxe01

[dhcp_servers]
pxe01
```

Для SSH-ключа этого достаточно. Пароль `sudo` безопаснее запрашивать параметром
`--ask-become-pass` или хранить в Ansible Vault.

Не рекомендуется хранить в открытом inventory:

```ini
ansible_password=plain-text-password
ansible_become_password=plain-text-password
```

### Два DNS-сервера

```ini
[dns_servers]
pxe01 ansible_host=192.168.0.24 ansible_user=user
dns02 ansible_host=192.168.0.25 ansible_user=user

[nginx_servers]
pxe01

[tftp_servers]
pxe01

[iso_servers]
pxe01

[dhcp_servers]
pxe01
```

Для этого сценария также измените `dns_secondary_enabled` и
`dhcp_dns_servers` в общем vars-файле.

## Единый файл сетевых переменных

Все PXE playbooks подключают
[`playbooks/vars/pxe.yaml`](playbooks/vars/pxe.yaml) через `vars_files`.

Текущая конфигурация:

```yaml
pxe_server_address: 192.168.0.24

dhcp_subnet: 192.168.0.0
dhcp_netmask: 255.255.255.0
dhcp_range_start: 192.168.0.100
dhcp_range_end: 192.168.0.200
dhcp_broadcast_address: 192.168.0.255
dhcp_gateway: 192.168.0.1
dhcp_dns_servers:
  - "{{ pxe_server_address }}"
dhcp_tftp_server: "{{ pxe_server_address }}"
dhcp_interfacesv4: []

pxe_http_server: "{{ pxe_server_address }}"

domain: dark.side.com
dns_secondary_enabled: false
dns_trusted_networks:
  - 192.168.0.0/24
dns_reverse_zone: 0.168.192.in-addr.arpa
dns_reverse_network: 192.168.0
dns_zone_records:
  - name: pxe
    address: "{{ pxe_server_address }}"
```

### Что изменить для другой сети

Для перехода, например, на другую `/24` сеть согласованно измените:

1. `pxe_server_address`;
2. `dhcp_subnet` и `dhcp_netmask`;
3. границы `dhcp_range_start`/`dhcp_range_end`;
4. `dhcp_broadcast_address` и `dhcp_gateway`;
5. `dhcp_dns_servers`;
6. `dhcp_tftp_server` и `pxe_http_server`, если сервисы распределены;
7. `dns_trusted_networks`;
8. `dns_reverse_zone` с обратным порядком октетов сети;
9. `dns_reverse_network` в прямом порядке первых трёх октетов;
10. записи `dns_zone_records`;
11. адреса `ansible_host` в inventory.

Текущий шаблон reverse-зоны создаёт PTR из последнего октета и рассчитан на
IPv4 `/24`. Для `/16`, classless delegation или нескольких подсетей роль DNS
нужно расширять, а не только менять строку `dns_reverse_zone`.

### Выбор DHCP-интерфейса

Если сервер имеет один интерфейс, допустимо оставить:

```yaml
dhcp_interfacesv4: []
```

Для нескольких NIC рекомендуется явно указать интерфейс клиентского сегмента:

```yaml
dhcp_interfacesv4:
  - ens19
```

Имя определяется на сервере командой `ip address`, но само сетевое назначение
адреса роль не выполняет.

### Добавление secondary DNS

```yaml
dns_secondary_enabled: true
dhcp_dns_servers:
  - 192.168.0.24
  - 192.168.0.25
```

Порядок хостов в `[dns_servers]` имеет значение: первый primary, второй
secondary. При изменении DNS-записей увеличьте `dns_zone_serial` в vars-файле
или переопределите его через `--extra-vars`.

## Подготовка ISO

ISO-файлы намеренно исключены из Git правилом `*.iso`. Они должны находиться на
Ansible controller внутри `roles/iso/files` по путям, указанным в `iso_images`.

Текущий каталог ожидает:

```text
roles/iso/files/
├── centos/
│   └── CentOS-7-x86_64-DVD-1810.iso
├── ubuntu_server/
│   └── ubuntu-26.04-live-server-amd64.iso
└── ubuntu_desktop/
    └── ubuntu-24.04.3-desktop-amd64.iso
```

Роль последовательно обрабатывает образы. После успешной публикации она:

- размонтирует ISO, если сама его смонтировала;
- удаляет staging-копию из `/tmp/pxe-images`;
- создаёт `/var/www/html/<name>/.pxe-published`;
- при повторном запуске пропускает образ с marker-файлом.

Чтобы перепубликовать все образы:

```yaml
iso_force_republish: true
```

После обновления верните значение в `false`. Marker означает завершение
предыдущей публикации, но не сравнивает checksum исходного ISO.

## Проверка перед запуском

Проверка inventory:

```bash
ansible-inventory -i playbooks/inventory.ini --graph
```

Проверка SSH и Python на целевом узле:

```bash
ansible -i playbooks/inventory.ini pxe01 -m ping
```

Проверка `become`:

```bash
ansible -i playbooks/inventory.ini pxe01 \
  --become \
  --ask-become-pass \
  -m command \
  -a "whoami"
```

Проверка синтаксиса полного playbook:

```bash
ansible-playbook \
  -i playbooks/inventory.ini \
  playbooks/pxe.yaml \
  --syntax-check
```

Просмотр выбранных хостов:

```bash
ansible-playbook \
  -i playbooks/inventory.ini \
  playbooks/pxe.yaml \
  --list-hosts
```

## Запуск полного PXE-стека

Для одного `pxe01`:

```bash
ANSIBLE_CONFIG="$PWD/ansible.cfg" \
ansible-playbook \
  -i playbooks/inventory.ini \
  playbooks/pxe.yaml \
  --limit pxe01 \
  --ask-become-pass
```

Если `sudo` не требует пароля, `--ask-become-pass` можно не указывать.

Для распределённого сценария или primary/secondary DNS не ограничивайте запуск
только `pxe01`:

```bash
ansible-playbook -i playbooks/inventory.ini playbooks/pxe.yaml
```

Либо перечислите все необходимые узлы:

```bash
ansible-playbook \
  -i playbooks/inventory.ini \
  playbooks/pxe.yaml \
  --limit 'pxe01,dns02'
```

## Запуск отдельных компонентов

### Только DHCP

```bash
ansible-playbook -i playbooks/inventory.ini playbooks/dhcp.yaml
```

Изменяемые параметры: DHCP-подсеть, пул, gateway, DNS, TFTP server и NIC в
`playbooks/vars/pxe.yaml`. Подробности:
[`roles/dhcp/README.md`](roles/dhcp/README.md).

### Только DNS

```bash
ansible-playbook -i playbooks/inventory.ini playbooks/dns.yaml
```

Требуется группа `dns_servers`. Для одиночного режима достаточно одного хоста,
для secondary — двух. Подробности: [`roles/dns/README.md`](roles/dns/README.md).

### Только Nginx

```bash
ansible-playbook -i playbooks/inventory.ini playbooks/nginx.yaml
```

Требуется группа `nginx_servers`. Роль создаёт пустой HTTP publication root;
контент добавляет ISO-роль. Подробности:
[`roles/nginx/README.md`](roles/nginx/README.md).

### Только TFTP

```bash
ansible-playbook -i playbooks/inventory.ini playbooks/tftp.yaml
```

Требуется группа `tftp_servers` и доступ target-хоста к Syslinux URL. Роль сама
собирает bootloader, но kernel/initrd добавляет ISO-роль. Подробности:
[`roles/tftp/README.md`](roles/tftp/README.md).

### Только публикация ISO

```bash
ansible-playbook -i playbooks/inventory.ini playbooks/iso.yaml
```

Требуется группа `iso_servers`, локальные ISO, настроенные TFTP/HTTP каталоги и
`become`. Подробности: [`roles/iso/README.md`](roles/iso/README.md).

### Только QEMU/KVM

Добавьте группу:

```ini
[qemu_kvm_servers]
hypervisor01 ansible_host=192.168.0.20 ansible_user=user
```

Запустите:

```bash
ansible-playbook -i playbooks/inventory.ini playbooks/qemu_kvm.yaml
```

Подробности: [`roles/qemu_kvm/README.md`](roles/qemu_kvm/README.md).

## Порядок выполнения полного playbook

[`playbooks/pxe.yaml`](playbooks/pxe.yaml) импортирует автономные playbooks в
следующем порядке:

1. `dns.yaml` — DNS должен быть готов до регистрации и использования имён;
2. `nginx.yaml` — HTTP root создаётся до публикации установочных файлов;
3. `tftp.yaml` — собираются Syslinux и стартовое меню;
4. `iso.yaml` — публикуются kernel/initrd и установочный контент;
5. `dhcp.yaml` — DHCP включается последним, чтобы клиент не получил ссылку на
   незавершённую инфраструктуру.

Каждый импортированный playbook повторно загружает
`playbooks/vars/pxe.yaml`. Это позволяет запускать его как отдельно, так и в
составе полного сценария с одинаковой конфигурацией.

## Добавление новой операционной системы

### 1. Разместить ISO

```text
roles/iso/files/<system>/<image>.iso
```

### 2. Описать образ

Добавьте элемент в `iso_images` в [`roles/iso/defaults/main.yaml`](roles/iso/defaults/main.yaml)
или переопределите весь список в vars:

```yaml
iso_images:
  - name: my_linux
    title: My Linux
    source: my_linux/my-linux.iso
    kernel: path/inside/iso/vmlinuz
    initrd: path/inside/iso/initrd
    publish_iso: true
```

Поля:

| Поле | Обязательное | Назначение |
|---|---|---|
| `name` | да | имя каталогов TFTP/HTTP и marker |
| `title` | да | читаемое имя в Ansible output |
| `source` | да | путь относительно `roles/iso/files` |
| `kernel` | да | путь к kernel внутри смонтированного ISO |
| `initrd` | да | путь к initrd внутри смонтированного ISO |
| `publish_iso` | нет | копировать ISO как HTTP `installer.iso` |

Текущая роль всегда копирует всё содержимое смонтированного образа в HTTP root.
`publish_iso: true` дополнительно создаёт `installer.iso` для загрузчиков,
использующих параметр `url=`.

### 3. Добавить меню

В [`roles/tftp/templates/default.j2`](roles/tftp/templates/default.j2):

```text
label 5
menu label ^5. Install My Linux
kernel my_linux/vmlinuz
append initrd=my_linux/initrd ip=dhcp ...
```

Пути `kernel` и `initrd` в меню считаются относительно `BIOS/` или `EFIx64/`,
куда ISO-роль копирует одинаковые файлы.

### 4. Удалить marker или включить force

Для существующего `name`:

```yaml
iso_force_republish: true
```

### 5. Применить изменения

```bash
ansible-playbook -i playbooks/inventory.ini playbooks/tftp.yaml
ansible-playbook -i playbooks/inventory.ini playbooks/iso.yaml
```

## Создаваемые сервисы и файлы

| Компонент | Сервис | Основные управляемые файлы/каталоги |
|---|---|---|
| DHCP | `isc-dhcp-server` | `/etc/dhcp/dhcpd.conf`, `/etc/default/isc-dhcp-server` |
| DNS | `bind9` | `/etc/default/named`, `/etc/bind/named.conf.options`, `/etc/bind/named.conf.local`, `/etc/bind/zones/` |
| HTTP | `nginx` | `/etc/nginx/sites-available/pxe`, `/etc/nginx/sites-enabled/pxe`, `/var/www/html/` |
| TFTP | `tftpd-hpa` | `/etc/default/tftpd-hpa`, `/var/lib/tftpboot/` |
| Syslinux | нет отдельного сервиса | `/opt/syslinux/syslinux-6.04-pre2/`, boot-файлы в TFTP root |
| ISO publication | нет отдельного сервиса | `/tmp/pxe-images/`, `/mnt/pxe/`, TFTP/HTTP roots |
| QEMU/KVM | `libvirtd` | пакеты QEMU/libvirt и системная конфигурация дистрибутива |

Резервные копии:

- DHCP: `/opt/backups/dhcp`;
- DNS: `/opt/backups/dns`.

Backup-задачи используют timestamp и создают новые копии при каждом запуске.
Автоматическая ротация backup в проекте пока не реализована.

## Переменные и их приоритет

В проекте используются два основных уровня:

1. `roles/<role>/defaults/main.yaml` — переносимые значения по умолчанию;
2. `playbooks/vars/pxe.yaml` — конфигурация штатной PXE-топологии.

Поскольку `vars/pxe.yaml` подключён как `vars_files` на уровне play, его значения
переопределяют role defaults и inventory group variables. Для разового запуска
можно использовать extra vars — они имеют ещё более высокий приоритет:

```bash
ansible-playbook \
  -i playbooks/inventory.ini \
  playbooks/dhcp.yaml \
  --extra-vars 'dhcp_interfacesv4=["ens19"]'
```

Для устойчивой конфигурации изменяйте общий vars-файл; extra vars используйте
для временных экспериментов или CI.

## Сетевая и эксплуатационная модель

### DHCP

DHCP — критичный L2-сервис. Перед включением убедитесь, что выбранная сеть и пул
не конфликтуют с существующей адресацией. Роль не выполняет автоматическое
обнаружение другого DHCP-сервера.

### DNS

BIND слушает `ansible_host` текущего inventory-хоста. ACL рекурсии формируется
из `dns_trusted_networks`. Upstream resolvers задаются `dns_forwarders`.

### HTTP и TFTP

Nginx публикует каталог с `autoindex on`; аутентификация и TLS не настроены.
TFTP запускается с `--secure` и ограничивает доступ `tftp_root`.

### ISO и хранение

ISO копируются через Ansible controller, поэтому transfer зависит от SSH и
свободного места на controller/target. Staging находится в `/tmp/pxe-images` и
удаляется после успешной публикации каждого образа. HTTP-копия репозитория и
`installer.iso` остаются постоянно и должны учитываться при планировании диска.

### Идемпотентность

Большинство задач использует декларативные Ansible-модули. Исключения и маркеры:

- Syslinux не пересобирается при наличии `bios/core/pxelinux.0`;
- ISO не перепубликуется при наличии `.pxe-published`, если
  `iso_force_republish` выключен;
- DNS zone transfer требует увеличения `dns_zone_serial`;
- backup-задачи намеренно создают timestamped копии и поэтому меняют состояние
  при каждом запуске.

## Документация ролей

- [DHCP](roles/dhcp/README.md)
- [DNS](roles/dns/README.md)
- [Nginx](roles/nginx/README.md)
- [TFTP и Syslinux](roles/tftp/README.md)
- [ISO publication](roles/iso/README.md)
- [QEMU/KVM](roles/qemu_kvm/README.md)
- [Статус NFS/NTP и roadmap](docs/ROADMAP.md)

Траблшутинг намеренно не включён в этот документ и должен быть оформлен
отдельным руководством по диагностике после стабилизации ролей.
