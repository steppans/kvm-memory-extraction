# 1. Подготовка среды исследования

## Содержание

1. [Подготовка среды исследования](#1-подготовка-среды-исследования)
   - [Установка необходимого инструментария](#установка-необходимого-инструментария)
   - [Проверка возможности виртуализации](#проверка-возможности-виртуализации)
   - [Выбор инструмента для запуска](#выбор-инструмента-для-запуска)
2. [Запуск виртуальных машин с помощью QEMU/KVM](#2-запуск-виртуальных-машин-с-помощью-qemukvm)
   - [Создание виртуального диска для Ubuntu](#создание-виртуального-диска-для-ubuntu)
   - [Создание seed образа для cloud-init](#создание-seed-образа-для-cloud-init)
   - [UEFI файлы](#uefi-файлы)
   - [Запуск виртуальной машины](#запуск-виртуальной-машины)
     - [Debug](#debug)
     - [Init](#init)
     - [Develop](#develop)
     - [Основные параметры QEMU](#основные-параметры-qemu)
   - [Подключение к виртуальной машине](#подключение-к-виртуальной-машине)
3. [Шифрование диска с помощью LUKS](#3-шифрование-диска-с-помощью-luks)
   - [Проверка образа виртуального диска](#проверка-образа-виртуального-диска)
   - [Использование GuestFS для безопасного монтирования](#использование-guestfs-для-безопасного-монтирования)
   - [Создание зашифрованного диска LUKS](#создание-зашифрованного-диска-luks)
   - [Запуск VM с зашифрованным диском](#запуск-vm-с-зашифрованного-диска)


## Установка необходимого инструментария

```bash
sudo apt update
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils
```

> **Примечание**
>
> - `qemu-kvm` - основной гипервизор:
> - `libvirt-daemon-system` - системный сервис для управления виртуальными машинами через API libvirt.
> - `libvirt-clients` - клиентские утилиты командной строки для взаимодействия с libvirt (создание, запуск, остановка VM).
> - `bridge-utils` - набор инструментов для создания и настройки сетевых мостов, позволяющих гостевым ОС взаимодействовать с сетью хоста и внешней сетью.


## Проверка возможности виртуализации

Для корректной работы гипервизора KVM необходимо, чтобы процессор поддерживал аппаратную виртуализацию. На ARM-платформах это можно проверить следующим образом:

```bash
ls -l /dev/kvm
```

Если устройство `/dev/kvm` существует и доступно, то KVM установлен и ядро поддерживает виртуализацию.

Запуск QEMU с опцией `-enable-kvm` позволяет проверить работоспособность аппаратного ускорения и версию QEMU:

```bash
qemu-system-aarch64 -enable-kvm --version
```

Также полезно просмотреть системный журнал на наличие сообщений о KVM, что подтверждает корректную инициализацию модуля ядра:

```bash
dmesg | grep -i kvm
```

> **Примечание**
>
> На процессорах Intel или AMD аппаратная виртуализация проверяется командой:
> 
> ```bash
> egrep -c '(vmx|svm)' /proc/cpuinfo
> ```
>
> Интерпретация результата:
> - `0` - аппаратная виртуализация отсутствует, KVM не может использовать ускорение;
> - `>0` - аппаратная виртуализация поддерживается, можно использовать KVM для ускорения виртуальных машин.


## Выбор инструмента для запуска

QEMU предоставляет отдельные исполняемые файлы для разных архитектур, чтобы корректно эмулировать процессор и устройства гостевой системы:

- `qemu-system-x86_64` - 64-битные процессоры Intel/AMD
- `qemu-system-i386` - 32-битные процессоры Intel/AMD
- `qemu-system-aarch64` - 64-битные процессоры ARM
- `qemu-system-arm` - 32-битные процессоры ARM

Выбор конкретного инструмента зависит от архитектуры исследуемого устройства. В случае **Raspberry Pi 4** используется 64-битный ARM-процессор, поэтому для запуска виртуальных машин на этой платформе применяется инструмент `qemu-system-aarch64`.


## 2. Запуск виртуальных машин с помощью QEMU/KVM

### Создание виртуального диска для Ubuntu

В корневой папке пользователя лежит официальный образ Ubuntu ARM64:

```plaintext
images/
└── ubuntu
    └── noble-server-cloudimg-arm64.img
```

Для сохранения изменений будущей виртуальной машины создаём отдельный диск формата `qcow2`, используя базовый образ:

```bash
qemu-img create -f qcow2 --backing-format qcow2 -b ~/images/ubuntu/raw/noble-server-cloudimg-arm64.img ~/images/ubuntu/raw/ubuntu.qcow2 10G
```

```plaintext
Formatting '/home/steppans/images/ubuntu/ubuntu.qcow2', fmt=qcow2 cluster_size=65536 extended_l2=off compression_type=zlib size=10737418240 backing_file=/home/steppans/images/ubuntu/noble-server-cloudimg-arm64.img backing_fmt=qcow2 lazy_refcounts=off refcount_bits=16
```

Пояснение параметров:

- `-f qcow2` - формат создаваемого виртуального диска;
- `-b` - базовый образ, на основе которого создаётся новый диск;
- `--backing-format qcow2` - формат базового образа;
- `10G` - максимальный размер нового диска (qcow2 растёт динамически по мере записи данных).

### Создание seed образа для cloud-init

Сначала устанавливаем утилиты для работы с cloud-образами:

```bash
sudo apt update
sudo apt install cloud-image-utils
```

- `cloud-image-utils` - набор инструментов для создания и настройки cloud-образов, включая генерацию seed-образов для cloud-init.

Для настройки виртуальной машины используются файлы конфигурации cloud-init:

- [user-data.yaml](../ubuntu/raw/cloud-init/user-data.yaml)
- [meta-data.yaml](../ubuntu/raw/cloud-init/meta-data.yaml)

После подготовки конфигурации создаём seed-образ, который будет использоваться для инициализации виртуальной машины:

```bash
cd ~/images/ubuntu/raw/clooud-init/
cloud-localds seed.img user-data.yaml meta-data.yaml
```


### UEFI файлы

Для запуска виртуальной машины используются два pflash-файла, обеспечивающие работу UEFI на ARM:

- `/usr/share/AAVMF/AAVMF_CODE.fd` - основной **UEFI код** (firmware), предоставляемый пакетом AAVMF. Этот файл является **только для чтения** и содержит все инструкции для загрузки ARM-платформы, включая стандартный bootloader и UEFI интерфейс.

- `~/images/ubuntu/raw/qemu-vars/qemu-vm001-vars.fd` - **файл переменных UEFI** для конкретной виртуальной машины. Он содержит сохранённые настройки прошивки, например, состояние boot order, конфигурацию устройств и другие параметры. Для возможности записи при запуске VM ему выставляются права на запись (`chmod +w`).

  - Этот файл используется для сохранения состояния VM между запусками.  

Использование двух этих файлов позволяет **отделить неизменяемый код UEFI от пользовательских настроек VM**, обеспечивая безопасность и воспроизводимость экспериментов.

Перед запуском создаём копию переменных UEFI для каждой VM и делаем её доступной для записи:

```bash
mkdir -p ~/images/ubuntu/raw/qemu-vars
cp /usr/share/AAVMF/AAVMF_VARS.fd ~/images/ubuntu/raw/qemu-vars/qemu-vm001-vars.fd
chmod +w ~/images/ubuntu/raw/qemu-vars/qemu-vm001-vars.fd
```

*Если есть ошибки с переписанным `AAVMF_VARS.fd`, то надо его восстановить*:

```bash
sudo apt install --reinstall qemu-efi-aarch64
```

### Запуск виртуальной машины

#### Debug

Проверка конфигурации seed-образа:

```bash
sudo qemu-system-aarch64 \
  -machine virt \
  -cpu host \
  -smp 2 \
  -m 2048 \
  -enable-kvm \
  -drive if=none,file=/home/steppans/images/ubuntu/raw/noble-server-cloudimg-arm64.img,id=hd0 \
  -device virtio-blk-device,drive=hd0 \
  -drive if=virtio,format=raw,file=/home/steppans/images/ubuntu/raw/cloud-init/seed.img \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -device virtio-net-device,netdev=net0 \
  -drive if=pflash,format=raw,file=/usr/share/AAVMF/AAVMF_CODE.fd,readonly=on \
  -drive if=pflash,format=raw,file=/home/steppans/images/ubuntu/raw/qemu-vars/qemu-vm001-vars.fd \
  -snapshot \
  -nographic
```

- Используется для проверки и отладки конфигурации VM.  
- `-snapshot` - все изменения диска теряются после выключения.  
- `-nographic` - Вывод идёт в текущий терминал.  

#### Init

Первый запуск VM с cloud-init:

```bash
sudo qemu-system-aarch64 \
  -machine virt \
  -cpu host \
  -smp 2 \
  -m 2048 \
  -enable-kvm \
  -drive if=none,file=/home/steppans/images/ubuntu/raw/ubuntu.qcow2,id=hd0 \
  -device virtio-blk-device,drive=hd0 \
  -drive if=virtio,format=raw,file=/home/steppans/images/ubuntu/raw/cloud-init/seed.img \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -device virtio-net-device,netdev=net0 \
  -drive if=pflash,format=raw,file=/usr/share/AAVMF/AAVMF_CODE.fd,readonly=on \
  -drive if=pflash,format=raw,file=/home/steppans/images/ubuntu/raw/qemu-vars/qemu-vm001-vars.fd \
  -display none \
  -daemonize
```

- Первый запуск VM с seed-образом для автоматической настройки пользователей и сети.  
- VM запускается в фоне, изменения диска сохраняются.  

#### Develop

Обычный запуск инициализированной VM:

```bash
sudo qemu-system-aarch64 \
  -machine virt \
  -cpu host \
  -smp 2 \
  -m 2048 \
  -enable-kvm \
  -drive if=none,file=/home/steppans/images/ubuntu/raw/ubuntu.qcow2,id=hd0 \
  -device virtio-blk-device,drive=hd0 \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -device virtio-net-device,netdev=net0 \
  -drive if=pflash,format=raw,file=/usr/share/AAVMF/AAVMF_CODE.fd,readonly=on \
  -drive if=pflash,format=raw,file=/home/steppans/images/ubuntu/raw/qemu-vars/qemu-vm001-vars.fd \
  -display none \
  -daemonize
```

- Обычный запуск уже инициализированной VM без cloud-init.  
- Используется собственный pflash vars-файл для сохранения настроек UEFI.  

#### Основные параметры QEMU

| Опция                                                        | Пояснение                                                               |
| ------------------------------------------------------------ | ----------------------------------------------------------------------- |
| `-machine virt`                                              | Эмулирует универсальную ARM-платформу *virt*, оптимальную для QEMU/KVM  |
| `-cpu host`                                                  | Передаёт гостю почти все возможности реального CPU                      |
| `-smp 2`                                                     | Количество виртуальных CPU                                              |
| `-m 2048`                                                    | Объём RAM в мегабайтах                                                  |
| `-enable-kvm`                                                | Включает аппаратную виртуализацию KVM                                   |
| `-drive if=none,file=...,id=hd0`                             | Определяет виртуальный диск без привязки к интерфейсу                   |
| `-device virtio-blk-device,drive=hd0`                        | Подключает диск к гостю как VirtIO-устройство (быстрее SATA/SCSI)       |
| `-drive if=virtio,file=seed.img,format=raw`                  | Передаёт seed.img как virtio-диск (cloud-init NoCloud datasource)       |
| `-netdev user,id=net0,hostfwd=tcp::2222-:22`                 | User-NAT сеть + проброс порта 2222→22 (SSH)                             |
| `-device virtio-net-device,netdev=net0`                      | Подключает сетевой адаптер VirtIO                                       |
| `-drive if=pflash,format=raw,file=AAVMF_CODE.fd,readonly=on` | Подключает UEFI-ПЗУ для ARM (OVMF/AAVMF boot firmware)                  |
| `-drive if=pflash,format=raw,file=qemu-vm001-vars.fd`        | Используется для хранения настроек UEFI VM                              |
| `-nographic`                                                 | Отключает графический вывод, консоль - в текущем терминале              |
| `-display none`                                              | Полностью отключает графический интерфейс (без консоли)                 |
| `-daemonize`                                                 | Запускает QEMU в фоне (демонизация процесса)                            |
| `-snapshot`                                                  | Диск работает только в RAM, все изменения теряются после выключения     |

### Подключение к виртуальной машине

```bash
ssh -p 2222 ubuntu@127.0.0.1
```


## 3. Шифрование диска с помощью LUKS

### Проверка образа виртуального диска

Перед шифрованием полезно убедиться, что виртуальный диск доступен и корректно читается. Для этого используется `qemu-img`.

```bash
qemu-img info ~/images/ubuntu/raw/ubuntu.qcow2
```

- Если диск занят запущенной VM, может появиться ошибка:

```plaintext
qemu-img: Could not open 'images/ubuntu/ubuntu.qcow2': Failed to get shared "write" lock
Is another process using the image [images/ubuntu/ubuntu.qcow2]?
```

- Это нормальное поведение: QCOW2-диски блокируются QEMU для предотвращения одновременной записи.

Для проверки свободного образа:

```bash
cp ~/images/ubuntu/raw/ubuntu.qcow2 /tmp/ubuntu.qcow2
qemu-img info /tmp/ubuntu.qcow2
```

```plaintext
image: /home/steppans/images/ubuntu/raw/ubuntu.qcow2
file format: qcow2
virtual size: 10 GiB (10737418240 bytes)
disk size: 72 MiB
cluster_size: 65536
backing file: /home/steppans/images/ubuntu/raw/noble-server-cloudimg-arm64.img
backing file format: qcow2
Format specific information:
    compat: 1.1
    compression type: zlib
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
    extended l2: false
Child node '/file':
    filename: /home/steppans/images/ubuntu/raw/ubuntu.qcow2
    protocol type: file
    file length: 72.1 MiB (75563008 bytes)
    disk size: 72 MiB
```

- Вывод содержит информацию о формате (`file format: qcow2`), размере, кластере, базовом образе (`backing file`) и других свойствах.
- Если диск не зашифрован, к нему можно подключиться и просмотреть содержимое.

### Использование GuestFS для безопасного монтирования

GuestFS предоставляет изолированную среду для работы с образами виртуальных машин, без запуска самой VM.

```bash
sudo guestfish --ro -a ~/images/ubuntu/raw/ubuntu.qcow2 -i
```

- `--ro` - подключение в режиме **read-only**, чтобы не повредить образ.  
- `-a` - добавление образа диска для работы.  
- `-i` - автоматическое обнаружение и монтирование ОС внутри образа.

Пример работы внутри оболочки GuestFS:

```bash
><fs> cat /home/ubuntu/date.info
Wed Dec 3 04:13:08 UTC 2025
```

- Этот инструмент позволяет безопасно извлекать файлы и просматривать структуру диска перед шифрованием.

### Создание зашифрованного диска LUKS

На данном этапе выполняется создание **нового виртуального диска**, предназначенного для установки операционной системы с включённым полным шифрованием. Такой подход необходим, поскольку существующий QCOW2-образ с уже установленной системой нельзя корректно «обернуть» в LUKS - шифрование применяется на этапе установки ОС.

Для установки с шифрованием диска используется официальный установочный `.iso` образ Ubuntu с автоматической конфигурацией (`autoinstall`) через cloud-init.

Файлы конфигурации cloud-init, используемые при установке:

- [`user-data.yaml`](../ubuntu/encrypted/cloud-init/user-data.yaml)
- [`meta-data.yaml`](../ubuntu/encrypted/cloud-init/meta-data.yaml)

Для задания пароля пользователя в файле `user-data.yaml` в cloud-config используется заранее сгенерированный хэш:

```bash
mkpasswd --method=SHA-512 --rounds=4096 123
```

Создание пустого QCOW2-образа, который будет использоваться установщиком Ubuntu:

```bash
qemu-img create -f qcow2 /home/steppans/images/ubuntu/encrypted/ubuntu-encrypted.qcow2 10G
```

На основе файлов конфигурации создаётся seed-образ, передаваемый виртуальной машине при запуске:

```bash
cd ~/images/ubuntu/encrypted/clooud-init/
cloud-localds seed.img user-data.yaml meta-data.yaml
```

При использовании UEFI на ARM каждая виртуальная машина должна иметь собственный файл переменных прошивки:

```bash
mkdir -p ~/images/ubuntu/encrypted/qemu-vars
cp /usr/share/AAVMF/AAVMF_VARS.fd ~/images/ubuntu/encrypted/qemu-vars/qemu-vm001-vars.fd  
chmod +w ~/images/ubuntu/encrypted/qemu-vars/qemu-vm001-vars.fd
```

Ниже приведена команда запуска виртуальной машины для первичной установки Ubuntu Server на заранее созданный QCOW2-диск с включённым полным шифрованием. Установка выполняется в автоматическом режиме с использованием cloud-init: `autoinstall`.

В данном запуске виртуальная машина загружается с установочного ISO-образа, а все параметры установки (включая шифрование root-раздела) задаются через seed-образ cloud-init.


```bash
sudo qemu-system-aarch64 \
  -machine virt \
  -cpu host \
  -m 4096 \
  -smp 4 \
  -enable-kvm \
  \
  -device virtio-scsi-pci,id=scsi0 \
  -drive if=none,id=cdrom,file=/home/steppans/images/ubuntu/encrypted/ubuntu-24.04.3-live-server-arm64.iso,media=cdrom,readonly=on \
  -device scsi-cd,drive=cdrom \
  \
  -drive if=none,id=hd0,file=/home/steppans/images/ubuntu/encrypted/ubuntu-encrypted.qcow2 \
  -device virtio-blk-device,drive=hd0 \
  \
  -drive if=virtio,format=raw,file=/home/steppans/images/ubuntu/encrypted/cloud-init/seed.img \
  \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -device virtio-net-device,netdev=net0 \
  \
  -drive if=pflash,format=raw,file=/usr/share/AAVMF/AAVMF_CODE.fd,readonly=on \
  -drive if=pflash,format=raw,file=/home/steppans/images/ubuntu/encrypted/qemu-vars/qemu-vm001-vars.fd \
  -nographic -serial mon:stdio
```

В данном запуске используются параметры, которые ранее не применялись:

- `-device virtio-scsi-pci,id=scsi0` - добавляет контроллер SCSI с поддержкой VirtIO. Используется для корректного подключения CD-ROM устройства в ARM-среде и соответствует рекомендациям для установочных ISO.

- `-drive if=none,id=cdrom,file=...,media=cdrom,readonly=on` - подключает установочный ISO-образ как виртуальный CD-ROM в режиме только на чтение.

- `-device scsi-cd,drive=cdrom` - экспортирует ISO-образ гостевой системы как SCSI CD-ROM устройство, с которого выполняется загрузка установщика.

- `-serial mon:stdio` - объединяет вывод последовательной консоли гостевой системы и монитор QEMU в стандартный поток ввода-вывода.

### Запуск VM с зашифрованного диска

После установки можно запуститься с зашифрованного диска:

```bash
sudo qemu-system-aarch64 \
  -machine virt \
  -cpu host \
  -smp 2 \
  -m 2048 \
  -enable-kvm \
  -drive if=none,id=hd0,file=/home/steppans/images/ubuntu/encrypted/ubuntu-encrypted.qcow2,format=qcow2 \
  -device virtio-blk-device,drive=hd0 \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -device virtio-net-device,netdev=net0 \
  -drive if=pflash,format=raw,file=/usr/share/AAVMF/AAVMF_CODE.fd,readonly=on \
  -drive if=pflash,format=raw,file=/home/steppans/images/ubuntu/encrypted/qemu-vars/qemu-vm001-vars.fd \
  -nographic -serial mon:stdio
```

> **Примечание**
>
> После установки Ubuntu с шифрованным root-разделом, для однократного запуска VM вручную иногда приходится редактировать строку ядра в GRUB при загрузки системы:
>
> ```conf
> linux /vmlinuz-6.8.0-88-generic root=/dev/mapper/ubuntu--vg-ubuntu--lv ro console=ttyAMA0,115200n8 cryptdevice=/dev/vda3:ubuntu-root
> ```
>
> - `console=ttyAMA0,115200n8` - основной tty для QEMU ARM;  
> - `cryptdevice=/dev/vda3:ubuntu-root` - указывает ядру, что root зашифрован.
>
> При успешной загрузке вывод в терминале должен быть примерно таким:
>
> ```
> Begin: Running /scripts/init-premount ... done.
> Begin: Mounting root file system ... Begin: Running /scripts/local-top ... Please unlock disk dm_crypt-0:
> ```
>
> Это значит, что загрузка в систему прошла успешно и устройство просит ввести парольную фразу для расшифровки root-раздела.
>
> Для того чтобы не менять GRUB вручную при каждом запуске, конфигурация сохраняется в `/etc/default/grub`:
>
> ```
> GRUB_CMDLINE_LINUX_DEFAULT="console=ttyAMA0,115200n8 cryptdevice=/dev/vda3:ubuntu-root"
> ```
>
> После редактирования:
>
> ```
> sudo update-grub
> ```
>
> В нашем случае, при использовании **cloud-init (`user-data.yaml`)** эта строка уже прописана автоматически, поэтому ручная корректировка GRUB не требуется. VM сразу загружается с зашифрованным root-разделом.
