# Linux OS Recovery Guide

Полное руководство по восстановлению Linux-системы из резервной копии (`backup.tar.gz`) с использованием **LIVE-режима (Live USB / Live CD)**.

Инструкция включает:

- создание GPT-разметки;
- создание разделов EFI, Boot, Swap и Root;
- форматирование разделов;
- восстановление файловой системы из резервной копии;
- настройку `fstab`;
- установку загрузчика **GRUB**;
- подготовку системы к загрузке.

> [!WARNING]
> Все действия полностью удаляют данные на выбранном диске. Перед выполнением убедитесь, что выбран правильный диск (`/dev/sdX`).

---

# Содержание

- [Требования](#требования)
- [1. Загрузка в LIVE-режим](#1-загрузка-в-live-режим)
- [2. Получение прав root](#2-получение-прав-root)
- [3. Установка необходимых пакетов](#3-установка-необходимых-пакетов)
- [4. Определение диска](#4-определение-диска)
- [5. Создание GPT и разделов](#5-создание-gpt-и-разделов)
- [6. Форматирование разделов](#6-форматирование-разделов)
- [7. Монтирование разделов](#7-монтирование-разделов)
- [8. Восстановление файлов](#8-восстановление-файлов)
- [9. Подготовка chroot](#9-подготовка-chroot)
- [10. Настройка системы](#10-настройка-системы)
- [11. Установка GRUB](#11-установка-grub)
- [12. Проверка](#12-проверка)
- [13. Завершение восстановления](#13-завершение-восстановления)
- [Рекомендуемая структура разделов](#рекомендуемая-структура-разделов)
- [License](#license)

---

# Требования

Необходимо иметь:

- Live USB / Live CD (Ubuntu, Debian, Alpine Linux и др.);
- резервную копию системы (`backup.tar.gz`);
- доступ к Интернету (при необходимости установки пакетов);
- систему с UEFI (инструкция рассчитана на GPT + UEFI).

---

# 1. Загрузка в LIVE-режим

Загрузитесь с **Live USB / Live CD** и откройте терминал.

---

# 2. Получение прав root

```bash
sudo -i
```

---

# 3. Установка необходимых пакетов

При необходимости установите отсутствующие утилиты.

## Debian / Ubuntu

```bash
apt update

apt install \
    dosfstools \
    grub-efi-amd64 \
    efibootmgr \
    nfs-common
```

## Alpine Linux

```bash
apk update

apk add \
    dosfstools \
    grub \
    grub-efi \
    efibootmgr \
    nfs-utils
```

---

# 4. Определение диска

Просмотреть список дисков:

```bash
lsblk
```

Выбрать диск для восстановления:

```bash
fdisk /dev/sdX
```

> Где `/dev/sdX` — диск, который будет полностью очищен.

---

# 5. Создание GPT и разделов

Создать новую GPT-таблицу:

```
g
```

## EFI-раздел (1 ГБ)

```
n
1
<Enter>
+1G

t
1
```

Тип:

```
EFI System (code 1)
```

---

## Boot-раздел (2 ГБ)

```
n
2
<Enter>
+2G

t
2
20
```

Тип:

```
Linux filesystem (8300)
```

---

## Swap-раздел (8 ГБ)

```
n
3
<Enter>
+8G

t
3
19
```

Тип:

```
Linux swap (8200)
```

---

## Root-раздел (оставшееся место)

```
n
4
<Enter>
<Enter>

t
4
20
```

Тип:

```
Linux filesystem (8300)
```

---

Сохранить изменения:

```
w
```

---

# 6. Форматирование разделов

## EFI

```bash
mkfs.fat -F32 /dev/sdX1
```

## Boot

```bash
mkfs.ext4 /dev/sdX2
```

## Swap

```bash
mkswap /dev/sdX3
swapon /dev/sdX3
```

## Root

```bash
mkfs.ext4 /dev/sdX4
```

---

# 7. Монтирование разделов

```bash
mkdir /mnt/linux

mount /dev/sdX4 /mnt/linux

mkdir /mnt/linux/boot
mount /dev/sdX2 /mnt/linux/boot

mkdir /mnt/linux/boot/efi
mount /dev/sdX1 /mnt/linux/boot/efi
```

---

# 8. Восстановление файлов

```bash
cd /mnt/linux

tar -xvpzf /path/to/backup.tar.gz \
    --numeric-owner \
    -C /mnt/linux
```

---

# 9. Подготовка chroot

Создать необходимые каталоги:

```bash
mkdir /mnt/linux/{proc,sys,dev,run,tmp}
```

Установить права:

```bash
chmod 555 /mnt/linux/proc
chmod 555 /mnt/linux/sys
chmod 1777 /mnt/linux/tmp
```

Подмонтировать системные файловые системы:

```bash
mount --rbind /dev /mnt/linux/dev
mount --rbind /proc /mnt/linux/proc
mount --rbind /sys /mnt/linux/sys
```

Войти в восстановленную систему:

```bash
chroot /mnt/linux /bin/bash
```

---

# 10. Настройка системы

Получить UUID:

```bash
blkid | grep -E "sdX1|sdX2|sdX3|sdX4"
```

Отредактировать файл:

```bash
nano /etc/fstab
```

Пример:

```fstab
# Root
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx / ext4 errors=remount-ro 0 1

# Boot
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /boot ext4 defaults 0 2

# EFI
UUID=XXXX-XXXX /boot/efi vfat defaults 0 2

# Swap
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx none swap sw 0 0
```

---

# 11. Установка GRUB

Установить загрузчик:

```bash
grub-install \
    --target=x86_64-efi \
    --efi-directory=/boot/efi \
    --bootloader-id=GRUB
```

Создать конфигурацию:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

Обновить initramfs.

### Debian / Ubuntu

```bash
update-initramfs -u
```

### Alpine Linux

```bash
apk fix kernel-hooks
```

---

# 12. Проверка

Проверить запись загрузчика:

```bash
efibootmgr -v
```

Проверить файловые системы:

```bash
df -h
```

Проверить структуру разделов:

```bash
lsblk
```

---

# 13. Завершение восстановления

Выйти из chroot:

```bash
exit
```

Размонтировать:

```bash
umount /mnt/linux/sys/fs/cgroup/systemd 2>/dev/null
umount /mnt/linux/sys/fs/cgroup 2>/dev/null
umount /mnt/linux/sys 2>/dev/null
umount /mnt/linux/proc 2>/dev/null
umount /mnt/linux/dev 2>/dev/null
```

Или одной командой:

```bash
umount -R /mnt/linux
```

Перезагрузить систему:

```bash
reboot
```

---

# Рекомендуемая структура разделов

| Размер | Раздел | Тип | Файловая система | Точка монтирования |
|---------|--------|----------------|------------------|--------------------|
| 512 МБ – 1 ГБ | sda1 | EFI System | FAT32 | `/boot/efi` |
| 1–2 ГБ | sda2 | Linux | ext4 | `/boot` |
| 4–8 ГБ | sda3 | Linux swap | swap | `swap` |
| Остальное | sda4 | Linux | ext4 | `/` |

---

# Итоговая схема

```text
/dev/sdX1  → EFI (FAT32)
/dev/sdX2  → /boot (ext4)
/dev/sdX3  → swap
/dev/sdX4  → / (ext4)
```

---

# Результат

После выполнения инструкции система будет:

- восстановлена из резервной копии;
- иметь корректную GPT-разметку;
- содержать правильно настроенный `fstab`;
- иметь установленный загрузчик **GRUB**;
- быть готовой к загрузке в обычном режиме.

---

# License

This project is licensed under the **MIT License**.

See the [LICENSE](LICENSE) file for details.