# 2. Демонстрация безопасности данных при выключенной VM

## Содержание

2. [Демонстрация безопасности данных при выключенной VM](#демонстрация-безопасности-данных-при-выключенной-vm)
   - [Проверка зашифрованного образа с помощью GuestFS](#проверка-зашифрованного-образа-с-помощью-guestfs)
   - [Использование NBD для низкоуровневого доступа к образу](#использование-nbd-для-низкоуровневого-доступа-к-образу)
   - [Анализ зашифрованного раздела](#анализ-зашифрованного-раздела)


## Проверка зашифрованного образа с помощью GuestFS

Для безопасного анализа содержимого образа используется утилита `guestfish`. Она запускает изолированную среду на базе QEMU и позволяет работать с образами виртуальных машин без их монтирования в основной системе.

```bash
sudo guestfish --ro -a ~/images/ubuntu/encrypted/ubuntu-encrypted.qcow2 -i
```

При запуске утилита запрашивает пароль для расшифровки LUKS-раздела:

```plaintext
Enter key or passphrase ("/dev/sda3"):
```

После ввода корректной passphrase GuestFS успешно обнаруживает установленную систему:

``` plaintext
Welcome to guestfish, the guest filesystem shell for
editing virtual machine filesystems and disk images.

Type: ‘help’ for help on commands
      ‘man’ to read the manual
      ‘quit’ to quit the shell

Operating system: Ubuntu 24.04.3 LTS
/dev/disk/by-id/dm-uuid-LVM-RpHnVwN7Ojk0KipECmUzLMHWGAncahKkwKrUutALnVyYaQtil2yd2CIP6E6a2oDP mounted on /
/dev/disk/by-uuid/65105c86-2d9d-4fbc-942e-03114afcf2e0 mounted on /boot
/dev/disk/by-uuid/7379-1AD1 mounted on /boot/efi
```

## Использование NBD для низкоуровневого доступа к образу

Для демонстрации работы с образом на уровне ядра используется механизм Network Block Device (NBD). Он позволяет экспортировать QCOW2-образ как полноценное блочное устройство /dev/nbdX, с которым можно работать стандартными утилитами Linux.

Загрузка модуля NBD и подключение образа:

```bash
sudo modprobe nbd max_part=8
sudo qemu-nbd --connect=/dev/nbd0 ~/images/ubuntu/encrypted/ubuntu-encrypted.qcow2
```

После подключения проверяем структуру диска:

```bash
lsblk /dev/nbd0
```

```plaintext
NAME     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
nbd0      43:0    0   10G  0 disk
|-nbd0p1  43:1    0  538M  0 part
|-nbd0p2  43:2    0  1.8G  0 part
`-nbd0p3  43:3    0  7.7G  0 part
```

## Анализ зашифрованного раздела

После подключения зашифрованного QCOW2-образа через NBD можно напрямую работать с его разделами как с обычными блочными устройствами. Основной интерес представляет зашифрованный раздел `/dev/nbd0p3`.

Для наглядной демонстрации используется команда побайтового просмотра:

```bash
sudo hexdump -C /dev/nbd0p3 | head
```

```plaintext
00000000  4c 55 4b 53 ba be 00 02  00 00 00 00 00 00 40 00  |LUKS..........@.|
00000010  00 00 00 00 00 00 00 03  00 00 00 00 00 00 00 00  |................|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00000040  00 00 00 00 00 00 00 00  73 68 61 32 35 36 00 00  |........sha256..|
00000050  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000060  00 00 00 00 00 00 00 00  68 de 94 c7 60 7a d9 e0  |........h...`z..|
00000070  06 d1 65 58 6d e9 64 8e  54 ce e2 59 11 68 7c 3e  |..eXm.d.T..Y.h|>|
00000080  85 c5 c1 c8 a3 27 1c 12  11 45 aa 67 eb 95 51 1c  |.....'...E.g..Q.|
00000090  c0 a4 7a 45 28 10 2a 69  1e 1c c8 86 20 99 bf f9  |..zE(.*i.... ...|
```

- Первые байты 4c 55 4b 53 в ASCII соответствуют строке LUKS - это сигнатура заголовка LUKS. Это означает, что раздел действительно является LUKS-контейнером, а не обычной файловой системой (ext4, xfs и т.д.).

- За пределами заголовка виден хаотичный набор байт, что указывает на зашифрованные данные, не имеющие читаемой структуры. Во всем выводе `hexdump` достаточно тяжело найти читаемые строки.

Таким образом, даже при прямом доступе к блочному устройству невозможно получить содержимое файлов без знания ключа.

Для просмотра структуры LUKS-заголовка используется утилита `cryptsetup`:

```bash
sudo cryptsetup luksDump /dev/nbd0p3
```

```
LUKS header information
Version:        2
Epoch:          3
Metadata area:  16384 [bytes]
Keyslots area:  16744448 [bytes]
UUID:           4abe98e5-2c1d-48d6-b09e-ea7f8b2533d9
Label:          (no label)
Subsystem:      (no subsystem)
Flags:          (no flags)

Data segments:
  0: crypt
        offset: 16777216 [bytes]
        length: (whole device)
        cipher: aes-xts-plain64
        sector: 512 [bytes]

Keyslots:
  0: luks2
        Key:        512 bits
        Priority:   normal
        Cipher:     aes-xts-plain64
        Cipher key: 512 bits
        PBKDF:      argon2id
        Time cost:  18
        Memory:     83350
        Threads:    4
        Salt:       37 4b 64 e0 7d 61 ad 0f b1 21 e0 25 0a f3 94 b3
                    55 85 18 ac ef c1 ab 40 f8 64 d5 32 03 0f a8 9d
        AF stripes: 4000
        AF hash:    sha256
        Area offset:32768 [bytes]
        Area length:258048 [bytes]
        Digest ID:  0
Tokens:
Digests:
  0: pbkdf2
        Hash:       sha256
        Iterations: 31089
        Salt:       2e a8 df 71 f6 15 90 95 ac a5 11 3e 6d c3 26 54
                    28 f4 51 9f 2f 6a 6a 21 17 bc e1 20 f9 3f c2 6d
        Digest:     a4 b5 ce b7 93 fa 1a 68 7c d7 b2 9a 2d 84 4f 4a
                    d5 d7 45 a7 c1 42 7e 89 fa b8 ce fc e4 82 6b 6d
```

Вывод содержит подробную информацию о конфигурации шифрования:

- `Version: 2` - используется современный формат LUKS2;
- `Cipher: aes-xts-plain64` - стандартный и рекомендуемый режим шифрования для дисков;
- `Key size: 512 bits` - используется 512-битный ключ (XTS использует два ключа AES-256);
- `PBKDF: argon2id` - современная функция выработки ключей, устойчивая к атакам на GPU и ASIC.

Для демонстрации принципа работы LUKS выполним ручное открытие зашифрованного раздела, экспортированного через NBD. Это позволяет наглядно показать, какие шаги необходимы для доступа к данным и почему прямое монтирование невозможно.

```bash
sudo cryptsetup open /dev/nbd0p3 luks-test
```

- Команда расшифровывает LUKS-контейнер `/dev/nbd0p3` после ввода корректной passphrase;
- В системе появляется новое устройство `/dev/mapper/luks-test`;
- На этом этапе происходит только расшифровка контейнера, но файловая система ещё недоступна напрямую.

```bash
sudo vgscan
sudo vgchange -ay
sudo lvdisplay
```

- `vgscan` - обнаруживает группы томов (Volume Groups);
- `vgchange -ay` - активирует найденные группы томов;
- `vdisplay` - отображает доступные логические тома.

После этого становятся доступны логические устройства вида:

- `/dev/ubuntu-vg/ubuntu-lv` - корневой раздел системы.

Монтирование файловой системы:

```bash
sudo mkdir -p /mnt/luks
sudo mount /dev/ubuntu-vg/ubuntu-lv /mnt/luks
```

После завершения анализа необходимо корректно размонтировать файловую систему и закрыть все уровни абстракции

```bash
sudo umount /mnt/luks
sudo vgchange -an
sudo cryptsetup close luks-test
```

> **Примечание**
>
> Для демонстрации защитного механизма LUKS выполним попытку прямого монтирования раздела без его расшифровки:
>
> ```bash
> sudo mount /dev/nbd0p3 /mnt/test
> ```
>
> ```plaintext
> mount: /mnt/test: unknown filesystem type 'crypto_LUKS'.
> ```
> 
> Ядро Linux распознаёт тип раздела как crypto_LUKS, а не как файловую систему.
> 
> Это означает, что:
> 
> - внутри нет ext4, xfs или другой ОС в открытом виде;
> - данные защищены криптографическим контейнером.
> 
> Без выполнения cryptsetup open доступ к содержимому невозможен.

После завершения анализа образ необходимо корректно отключить от NBD:

```bash
sudo qemu-nbd --disconnect /dev/nbd0
```
