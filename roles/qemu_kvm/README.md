# Роль `qemu_kvm`

Независимая от PXE роль для подготовки Linux-хоста виртуализации на QEMU/KVM и libvirt. Она полезна для создания тестовых виртуальных машин, в том числе PXE-клиентов, но не входит в `playbooks/pxe.yaml`.

## Что делает роль

1. обновляет индекс APT;
2. устанавливает QEMU, OVMF, сетевые и диагностические пакеты;
3. устанавливает libvirt и `virt-install`;
4. при необходимости устанавливает графические компоненты QEMU;
5. запускает `kvm-ok` и завершает play при отсутствии аппаратной поддержки KVM;
6. включает и запускает `libvirtd`.

Роль не создаёт виртуальные машины, storage pools, bridge-интерфейсы или libvirt networks.

## Требования

- Debian/Ubuntu-подобный физический сервер или VM с доступной nested virtualization;
- включённые Intel VT-x/AMD-V;
- APT, systemd и права `become`;
- inventory-группа `[qemu_kvm_servers]`.

## Переменные

| Переменная | Назначение |
|---|---|
| `qemu_kvm_packages` | QEMU, OVMF, bridge-utils, dnsmasq-base и cpu-checker |
| `qemu_kvm_libvirt_packages` | Демон, CLI-клиенты и `virtinst` |
| `qemu_kvm_install_gui` | Устанавливать ли пакеты из `qemu_kvm_gui_packages`; по умолчанию `false` |
| `qemu_kvm_gui_packages` | Графические компоненты; по умолчанию `qemu-system-gui` |
| `qemu_kvm_service_name` | Имя службы libvirt; по умолчанию `libvirtd` |

Для серверной установки обычно оставляют `qemu_kvm_install_gui: false`.

## Inventory и запуск

```ini
[qemu_kvm_servers]
hypervisor01 ansible_host=192.168.0.30 ansible_user=user
```

```bash
ANSIBLE_CONFIG="$PWD/ansible.cfg" ansible-playbook \
  -i playbooks/inventory.ini playbooks/qemu_kvm.yaml \
  --limit hypervisor01 --ask-become-pass
```

## Результат

После выполнения на узле доступны QEMU CLI, UEFI-прошивка OVMF, инструменты libvirt и работающая служба `libvirtd`. Дальнейшее проектирование виртуальных сетей и машин остаётся ответственностью пользователя или будущих ролей проекта.
