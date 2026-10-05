# Инструкция 4 · ESP в `/boot/efi` · 🔐 LUKS2 (шифрованный корень) · Secure Boot

**Стек:** GRUB (EFI) + sbctl + LUKS2 · **Разделов:** 3 · **Время:** 45–55 минут
**Где работает:** реальный компьютер (UEFI) и VirtualBox

> ⚠️ Все данные на выбранном диске будут уничтожены.
> ⚠️ **Пароль LUKS восстановить нельзя.** Забыли — данные потеряны. Запишите его в надёжном месте.

## Что получится

| Раздел | Размер | Тип / ФС | Точка монтирования | Зашифрован |
|--------|--------|----------|--------------------|------------|
| `/dev/sda1` | 512 МБ | EFI System / FAT32 | `/boot/efi` | ❌ нет |
| `/dev/sda2` | 1 ГБ | Linux filesystem / ext4 | `/boot` | ❌ нет |
| `/dev/sda3` | всё остальное | LUKS2 → ext4 | `/` (как `cryptroot`) | ✅ да |

Зачем три раздела: прошивка читает только ESP (`/boot/efi`), а GRUB должен прочитать ядро из `/boot`
**до** ввода пароля. Поэтому `/boot` остаётся незашифрованным, а всё остальное — система, программы,
домашние папки — лежит внутри LUKS. Пароль вводится **один раз**, при загрузке.

> 🔤 **Придумывайте пароль LUKS из латинских букв и цифр.** В ранней загрузке раскладка всегда
> английская, кириллический пароль ввести не получится.

---

## Шаг 1. VirtualBox: создать ВМ

> На реальном компьютере пропустите этот шаг и переходите к шагу 2.

- [ ] Скачать ISO: https://archlinux.org/download/
- [ ] Создать ВМ: тип **Linux → Arch Linux (64-bit)**, ОЗУ **4096 МБ**, диск **30 ГБ** (VDI, динамический)
- [ ] **Настройки → Система → Материнская плата**: ✅ **Включить EFI**;
      **Secure Boot — НЕ включать** (почему — см. шаг 8)
- [ ] **Настройки → Носители**: подключить скачанный ISO к виртуальному приводу
- [ ] **Настройки → Система → Порядок загрузки**: optical первым
- [ ] Запустить ВМ → в меню ISO выбрать **Arch Linux install medium**

*(Необязательно, для нормального размера экрана в ВМ после установки:
`pacman -S virtualbox-guest-utils` и `systemctl enable vboxservice`.)*

---

## Шаг 2. Live-ISO: разметка и шифрование 🔐

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
`New` → `512M` → `Type` → **EFI System** (это `/dev/sda1`) →
`New` → `1G` → `Type` → **Linux filesystem** (это `/dev/sda2`) →
`New` → всё оставшееся место → `Type` → **Linux filesystem** (это `/dev/sda3`) →
`Write` → `yes` → `Quit`

- [ ] 🔐 Зашифровать **третий** раздел (потребуется дважды ввести пароль LUKS; на вопрос ответить `YES`):

```bash
cryptsetup luksFormat --type luks2 /dev/sda3
cryptsetup open /dev/sda3 cryptroot
```

- [ ] Создать файловые системы и смонтировать (порядок важен) — **вариант A: ext4** (проще):

```bash
mkfs.fat -F32 /dev/sda1
mkfs.ext4 /dev/sda2
mkfs.ext4 /dev/mapper/cryptroot
mount /dev/mapper/cryptroot /mnt
mount --mkdir /dev/sda2 /mnt/boot
mount --mkdir /dev/sda1 /mnt/boot/efi
```

- [ ] **Или вариант B: btrfs** для зашифрованного корня — вместо варианта A:

```bash
mkfs.fat -F32 /dev/sda1
mkfs.ext4 /dev/sda2
mkfs.btrfs -L archroot /dev/mapper/cryptroot
mount /dev/mapper/cryptroot /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
umount /mnt
mount -o noatime,compress=zstd,subvol=@ /dev/mapper/cryptroot /mnt
mount --mkdir -o noatime,compress=zstd,subvol=@home /dev/mapper/cryptroot /mnt/home
mount --mkdir /dev/sda2 /mnt/boot
mount --mkdir /dev/sda1 /mnt/boot/efi
```

> 🌿 **Про btrfs:** `@` — это корень, `@home` — домашние папки (без подтомов не работают снапшоты).
> `genfstab` сам пропишет `subvol=` и `compress=zstd` в `/etc/fstab` — руками добавлять ничего не нужно.
> Swap-файл на btrfs не работает: если нужен swap, используйте `zram-generator` (вне рамок чек-листа).
> Снапшоты (`snapper`, `timeshift`) — тоже отдельная тема. Выполнять нужно **только один** вариант: A или B.

> 🌿 `/boot` в этой схеме **намеренно остаётся ext4**, а не btrfs: именно там лежат ядро и initramfs,
> а GRUB не умеет читать zstd-сжатые файлы на btrfs. Это самая безопасная комбинация.
> В варианте B в `/etc/fstab` будет четыре строки: `/`, `/home`, `/boot`, `/boot/efi`.

- [ ] Проверить, что смонтировались все разделы (вариант A: `/mnt`, `/mnt/boot`, `/mnt/boot/efi`; вариант B — ещё и `/mnt/home`):

```bash
findmnt --real
```

---

## Шаг 3. Установить систему

- [ ] Установить базовый набор пакетов (обратите внимание на `cryptsetup`):

```bash
pacstrap -K /mnt base linux linux-firmware intel-ucode amd-ucode cryptsetup \
  grub efibootmgr sbctl sudo nano networkmanager
```

- [ ] Записать таблицу монтирования и войти в новую систему:

```bash
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

- [ ] Проверить fstab: должно быть **три** строки (в варианте B — четыре) — `/dev/mapper/cryptroot` для `/`, а также `/boot` и `/boot/efi`:

```bash
cat /etc/fstab
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

## Шаг 4. 🔐 Научить систему открывать шифрованный диск

- [ ] Добавить хук `encrypt` в initramfs (после `block`, до `filesystems`):

```bash
sed -i 's/^HOOKS=.*/HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block encrypt filesystems fsck)/' /etc/mkinitcpio.conf
grep '^HOOKS' /etc/mkinitcpio.conf
```

- [ ] Вписать параметры ядра в GRUB (UUID подставится автоматически, ничего копировать не нужно):

```bash
LUKS_UUID=$(blkid -s UUID -o value /dev/sda3)
echo "UUID шифрованного раздела: $LUKS_UUID"
sed -i "s|^GRUB_CMDLINE_LINUX=.*|GRUB_CMDLINE_LINUX=\"cryptdevice=UUID=$LUKS_UUID:cryptroot root=/dev/mapper/cryptroot rw\"|" /etc/default/grub
grep '^GRUB_CMDLINE_LINUX' /etc/default/grub
```

Если во второй строке вместо UUID пусто — проверьте имя раздела командой `lsblk` (на ПК это
обычно `/dev/nvme0n1p3`) и повторите блок с правильным именем.

- [ ] Пересобрать initramfs:

```bash
mkinitcpio -P
```

---

## Шаг 5. Загрузчик GRUB

- [ ] Установить GRUB в ESP и создать конфиг:

```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=arch
grub-mkconfig -o /boot/grub/grub.cfg
```

- [ ] Проверить, что параметры шифрования попали в конфиг (строка должна содержать `cryptdevice=UUID=`):

```bash
grep cryptdevice /boot/grub/grub.cfg
```

- [ ] Убедиться, что файл загрузчика создан:

```bash
ls /boot/efi/EFI/arch/grubx64.efi
```

- [ ] Выйти, отмонтировать и перезагрузиться:

```bash
exit
umount -R /mnt
cryptsetup close cryptroot
reboot
```

> В VirtualBox: после `reboot` выключите ВМ и **отключите ISO** в Настройки → Носители,
> иначе снова загрузится установщик.

---

## Шаг 6. Первый запуск

- [ ] В меню GRUB выбрать **Arch Linux**
- [ ] Появится запрос пароля → ввести **пароль LUKS** (символы не отображаются — это нормально).
      Запрос должен появиться **один раз**.
- [ ] Система загрузилась → войти как `user` → `su -` (или `sudo -i`)
- [ ] Проверить, что диск действительно зашифрован, а разделы на месте:

```bash
lsblk
findmnt --real
```

Должно быть видно `cryptroot` (точка `/`), `/dev/sda2` на `/boot` и `/dev/sda1` на `/boot/efi`.

---

## Шаг 7. Secure Boot: создать ключи и подписать GRUB

- [ ] Посмотреть текущее состояние:

```bash
sbctl status
```

Ожидаемо: `Secure Boot: Disabled`.

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
sbctl sign -s /boot/efi/EFI/arch/grubx64.efi
```

- [ ] Проверить подпись:

```bash
sbctl verify
sbctl list-files
```

> В этой схеме `sbctl verify` должен пройти **чисто**: на ESP лежит только GRUB, а ядро и initramfs
> находятся в `/boot` на ext4-разделе, куда прошивка не заглядывает.

- [ ] Перезагрузиться: `reboot` → ввести пароль LUKS → система загружается как обычно

---

## Шаг 8. Включить Secure Boot

### На реальном компьютере

- [ ] Перезагрузка → войти в прошивку (`Del`, `F2` или `F10` при включении)
- [ ] Найти раздел **Secure Boot** → поставить **Enabled** → сохранить и выйти (`F10`)
- [ ] Ввести пароль LUKS → система загрузилась → проверить:

```bash
sbctl status
```

Должно быть: `Secure Boot: Enabled` и `sbctl installed: yes`. Готово ✅

### В VirtualBox

- [ ] Ничего включать не нужно — см. предупреждение ниже

> ⚠️ **Почему в VirtualBox Secure Boot не включаем.**
> VirtualBox умеет работать только с ключами Microsoft, а добавить свои ключи через интерфейс ВМ
> нельзя. Если поставить галочку «Enable Secure Boot», ВМ откажется загружать наш GRUB, подписанный
> вашим ключом. Шаг 7 при этом не «тренировочный»: ключи действительно созданы, а GRUB действительно
> подписан — в VirtualBox просто нет прошивки, которая приняла бы эти ключи.

---

## Шаг 9. Если что-то пошло не так

| Симптом | Что делать |
|---------|------------|
| После загрузки снова спрашивает пароль и не пускает | неверный пароль LUKS. Раскладка при загрузке английская, Caps Lock выключен? |
| «unknown filesystem» / система не нашла корень | ошибка в `GRUB_CMDLINE_LINUX`. Загрузитесь с ISO → `cryptsetup open /dev/sda3 cryptroot` → `mount /dev/mapper/cryptroot /mnt` → `mount /dev/sda2 /mnt/boot` → `mount /dev/sda1 /mnt/boot/efi` → `arch-chroot /mnt` → повторить шаги 4–5 |
| В меню GRUB нажать `e` | можно временно исправить строку `linux …` (правильный `cryptdevice=UUID=…`), нажать `F10` и загрузиться |
| ВМ попала в **UEFI Interactive Shell** | наберите `FS0:` ↵, затем `\EFI\arch\grubx64.efi` ↵ |
| После включения SB — «Access denied» / чёрный экран | выключите SB в прошивке, загрузитесь, выполните `sbctl verify` и переподпишите файл |
| `sbctl enroll-keys` пишет «Access denied» | прошивка не в Setup Mode — очистите ключи SB в BIOS (см. шаг 7) |
| Забыли пароль root (LUKS помните) | загрузитесь с ISO → откройте и смонтируйте (см. строку выше) → `arch-chroot /mnt` → `passwd` |
| GRUB не видит диск / неверное имя | проверьте `lsblk`; на ПК диск обычно `/dev/nvme0n1`, разделы — `p1`, `p2`, `p3` |

---

## После установки (коротко)

- Обновление системы: `sudo pacman -Syu`. После обновления **GRUB переподписывается автоматически**
  (sbctl ставит pacman-hook). Проверить: `sbctl list-files`.
- Поставили второе ядро? Пересоберите меню: `sudo grub-mkconfig -o /boot/grub/grub.cfg`.
- 🔐 **Хотите зашифровать и `/boot`?** Тогда GRUB придётся самому открывать LUKS — это уровень
  «продвинуто»: нужен `GRUB_ENABLE_CRYPTODISK=y`, пароль вводится **дважды**, а контейнер обязан
  быть создан с `cryptsetup luksFormat --type luks2 --pbkdf pbkdf2` (GRUB не понимает Argon2).
  Подробности: ArchWiki → **GRUB#Encrypted /boot** и **dm-crypt/Device encryption**.
- 🔐 Вход без пароля через TPM2 (`systemd-cryptenroll`) — тоже отдельная тема, в чек-лист не входит.
- Дальше: ArchWiki → **dm-crypt** и **General recommendations**.
