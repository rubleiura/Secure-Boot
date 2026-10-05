# Инструкция 1 · ESP в `/boot` · простая разметка · Secure Boot

**Стек:** GRUB (EFI) + sbctl · **Разделов:** 2 · **Время:** 30–40 минут
**Где работает:** реальный компьютер (UEFI) и VirtualBox

> ⚠️ Все данные на выбранном диске будут уничтожены. Перед разметкой выполните `lsblk` и убедитесь,
> что диск правильный.

## Что получится

| Раздел | Размер | Тип / ФС | Точка монтирования |
|--------|--------|----------|--------------------|
| `/dev/sda1` | 1 ГБ | EFI System / FAT32 | `/boot` |
| `/dev/sda2` | всё остальное | Linux filesystem / ext4 | `/` |

Особенность схемы: ESP и есть `/boot`, поэтому ядро и initramfs лежат прямо на FAT-разделе.
Swap не создаём — для простоты.

---

## Шаг 1. VirtualBox: создать ВМ

> На реальном компьютере пропустите этот шаг и переходите к шагу 2.

- [ ] Скачать ISO: https://archlinux.org/download/
- [ ] Создать ВМ: тип **Linux → Arch Linux (64-bit)**, ОЗУ **4096 МБ**, диск **30 ГБ** (VDI, динамический)
- [ ] **Настройки → Система → Материнская плата**: ✅ **Включить EFI**;
      **Secure Boot — НЕ включать** (почему — см. шаг 7)
- [ ] **Настройки → Носители**: подключить скачанный ISO к виртуальному приводу
- [ ] **Настройки → Система → Порядок загрузки**: optical первым
- [ ] Запустить ВМ → в меню ISO выбрать **Arch Linux install medium**

*(Необязательно, для нормального размера экрана в ВМ после установки:
`pacman -S virtualbox-guest-utils` и `systemctl enable vboxservice`.)*

---

## Шаг 2. Live-ISO: проверить режим и разметить диск

- [ ] Загрузились в live-систему, открылась командная строка `root@archiso`
- [ ] Проверить, что загрузка в UEFI-режиме (должно напечатать `64`):

```bash
cat /sys/firmware/efi/fw_platform_size
```

- [ ] (Необязательно) русская раскладка и шрифт в консоли:

```bash
loadkeys ru
setfont cyr-sun16
```

- [ ] Проверить интернет:

```bash
ping -c 2 archlinux.org
```

- [ ] Найти свой диск и разметить его:

```bash
lsblk
cfdisk /dev/sda
```

В `cfdisk`: выбрать **gpt** →
`New` → `1G` → `Type` → **EFI System** →
`New` → всё оставшееся место → `Type` → **Linux filesystem** →
`Write` → `yes` → `Quit`

- [ ] Создать файловые системы и смонтировать — **вариант A: ext4** (проще, рекомендуется для первой установки):

```bash
mkfs.fat -F32 /dev/sda1
mkfs.ext4 /dev/sda2
mount /dev/sda2 /mnt
mount --mkdir /dev/sda1 /mnt/boot
```

- [ ] **Или вариант B: btrfs** (сжатие zstd и снапшоты) — вместо варианта A:

```bash
mkfs.fat -F32 /dev/sda1
mkfs.btrfs -L archroot /dev/sda2
mount /dev/sda2 /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
umount /mnt
mount -o noatime,compress=zstd,subvol=@ /dev/sda2 /mnt
mount --mkdir -o noatime,compress=zstd,subvol=@home /dev/sda2 /mnt/home
mount --mkdir /dev/sda1 /mnt/boot
```

> 🌿 **Про btrfs:** `@` — это корень, `@home` — домашние папки (без подтомов не работают снапшоты).
> `genfstab` сам пропишет `subvol=` и `compress=zstd` в `/etc/fstab` — руками добавлять ничего не нужно.
> Swap-файл на btrfs не работает: если нужен swap, используйте `zram-generator` (вне рамок чек-листа).
> Снапшоты (`snapper`, `timeshift`) — тоже отдельная тема. Выполнять нужно **только один** вариант: A или B.

- [ ] Проверить, что смонтировалось правильно (вариант A: `/mnt` и `/mnt/boot`; вариант B — ещё и `/mnt/home`):

```bash
findmnt --real
```

---

## Шаг 3. Установить систему

- [ ] Установить базовый набор пакетов:

```bash
pacstrap -K /mnt base linux linux-firmware intel-ucode amd-ucode \
  grub efibootmgr sbctl sudo nano networkmanager
```

- [ ] Записать таблицу монтирования и войти в новую систему:

```bash
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

- [ ] Время и локаль (часовой пояс замените на свой):

```bash
ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime
hwclock --systohc
sed -i 's/#ru_RU.UTF-8/ru_RU.UTF-8/; s/#en_US.UTF-8/en_US.UTF-8/' /etc/locale.gen
locale-gen
echo "LANG=ru_RU.UTF-8" > /etc/locale.conf
```

- [ ] Имя компьютера и сеть:

```bash
echo "arch-pc" > /etc/hostname
systemctl enable NetworkManager
```

- [ ] Пароль root и обычный пользователь (`user` замените на своё имя):

```bash
passwd
useradd -m -G wheel -s /bin/bash user
passwd user
echo "%wheel ALL=(ALL:ALL) ALL" > /etc/sudoers.d/wheel
chmod 440 /etc/sudoers.d/wheel
```

---

## Шаг 4. Загрузчик GRUB

- [ ] Установить GRUB в ESP и создать конфиг:

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=arch
grub-mkconfig -o /boot/grub/grub.cfg
```

- [ ] Убедиться, что файл загрузчика создан:

```bash
ls /boot/EFI/arch/grubx64.efi
```

- [ ] Выйти и перезагрузиться:

```bash
exit
umount -R /mnt
reboot
```

> В VirtualBox: после `reboot` выключите ВМ и **отключите ISO** в Настройки → Носители,
> иначе снова загрузится установщик.

---

## Шаг 5. Первый запуск

- [ ] В меню GRUB выбрать **Arch Linux** → система загрузилась
- [ ] Войти как `user`, получить права root: `su -` (или `sudo -i`)
- [ ] Проверить, что сеть работает: `ping -c 2 archlinux.org`

---

## Шаг 6. Secure Boot: создать ключи и подписать GRUB

- [ ] Посмотреть текущее состояние:

```bash
sbctl status
```

Ожидаемо: `Secure Boot: Disabled` (на ПК — потому что SB ещё выключен или нет ваших ключей,
в VirtualBox — потому что SB мы не включали).

- [ ] Создать свои ключи:

```bash
sbctl create-keys
```

- [ ] Записать ключи в прошивку (флаг `-m` сохраняет ключи Microsoft — нужен для видеокарты и dual-boot):

```bash
sbctl enroll-keys -m
```

> Если ошибка **«Access denied»** — прошивка не в режиме *Setup Mode*.
> Зайдите в BIOS/UEFI → раздел Secure Boot → **Clear / Delete Secure Boot keys**
> (или «Reset to Setup Mode») → загрузитесь в Arch → повторите команду.

- [ ] Подписать GRUB (флаг `-s` запоминает файл, чтобы переподписывать его автоматически):

```bash
sbctl sign -s /boot/EFI/arch/grubx64.efi
```

- [ ] Проверить подпись:

```bash
sbctl verify
sbctl list-files
```

> В этой схеме `sbctl verify` может сообщить, что `/boot/vmlinuz-linux` и `/boot/initramfs-linux.img`
> не подписаны. **Это нормально:** их загружает GRUB, а не прошивка. Подписан должен быть только
> `grubx64.efi` — он и есть в выводе `sbctl list-files`.

- [ ] Перезагрузиться: `reboot` → система должна загрузиться как обычно

---

## Шаг 7. Включить Secure Boot

### На реальном компьютере

- [ ] Перезагрузка → войти в прошивку (`Del`, `F2` или `F10` при включении)
- [ ] Найти раздел **Secure Boot** → поставить **Enabled** → сохранить и выйти (`F10`)
- [ ] Система загрузилась → проверить:

```bash
sbctl status
```

Должно быть: `Secure Boot: Enabled` и `sbctl installed: yes`. Готово ✅

### В VirtualBox

- [ ] Ничего включать не нужно — см. предупреждение ниже

> ⚠️ **Почему в VirtualBox Secure Boot не включаем.**
> VirtualBox умеет работать только с ключами Microsoft, а добавить свои ключи через интерфейс ВМ
> нельзя. Если поставить галочку «Enable Secure Boot», ВМ откажется загружать наш GRUB, подписанный
> вашим ключом. Шаг 6 при этом не «тренировочный»: ключи действительно созданы, а GRUB действительно
> подписан — в VirtualBox просто нет прошивки, которая приняла бы эти ключи.

---

## Шаг 8. Если что-то пошло не так

| Симптом | Что делать |
|---------|------------|
| ВМ попала в **UEFI Interactive Shell** | наберите `FS0:` ↵, затем `\EFI\arch\grubx64.efi` ↵ |
| После включения SB — «Access denied» / чёрный экран | выключите SB в прошивке, загрузитесь, выполните `sbctl verify` и переподпишите файл |
| `sbctl enroll-keys` пишет «Access denied» | прошивка не в Setup Mode — очистите ключи SB в BIOS (см. шаг 6) |
| Нет интернета после установки | `systemctl start NetworkManager`, затем `nmcli device wifi connect ИМЯ password ПАРОЛЬ` |
| Забыли пароль root | загрузитесь с ISO → `mount /dev/sda2 /mnt` → `arch-chroot /mnt` → `passwd` |
| GRUB не видит диск / неверное имя | проверьте `lsblk`; на ПК диск обычно `/dev/nvme0n1`, разделы — `p1`, `p2` |

---

## После установки (коротко)

- Обновление системы: `sudo pacman -Syu`. После обновления **GRUB переподписывается автоматически**
  (sbctl ставит pacman-hook). Проверить: `sbctl list-files`.
- Поставили второе ядро (например `linux-zen`)? Пересоберите меню: `sudo grub-mkconfig -o /boot/grub/grub.cfg`.
- Дальше: ArchWiki → **General recommendations** (пользователи, графика, звук).
