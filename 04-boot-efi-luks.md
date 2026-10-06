# Инструкция 4 · ESP в `/boot/efi` · 🔐 LUKS2 (шифрованный корень) · Secure Boot

**Стек:** systemd-boot + UKI + sbctl + LUKS2 · **Разделов:** 2 · **Время:** 45–55 минут
**Где работает:** реальный компьютер (UEFI) и VirtualBox

> ⚠️ Все данные на выбранном диске будут уничтожены.
> ⚠️ **Пароль LUKS восстановить нельзя.** Запишите его в надёжном месте.
> ⚠️ **Почему не GRUB:** при включённом Secure Boot GRUB из Arch отказывается грузить модули без shim
> (`shim_lock_verifier_init: prohibited by secure boot policy` → `grub rescue`). Здесь systemd-boot + UKI.

## Что получится

| Раздел | Размер | Тип / ФС | Точка монтирования | Зашифрован |
|--------|--------|----------|--------------------|------------|
| `/dev/sda1` | 1 ГБ | EFI System / FAT32 | `/boot/efi` | ❌ нет |
| `/dev/sda2` | всё остальное | LUKS2 → ext4 (или btrfs) | `/` (как `cryptroot`) | ✅ да |

`/boot` — папка **внутри зашифрованного корня**, там лежит только `vmlinuz-linux-lts`, и для загрузки
она не нужна. Ядро, initramfs и параметры командной строки упакованы в один подписанный файл **UKI**
на ESP: `/boot/efi/EFI/Linux/arch-linux-lts.efi`.

**Цепочка доверия:** прошивка → подписанный **systemd-boot** → подписанный **UKI** → initramfs
спрашивает пароль LUKS. Параметр `cryptdevice=…` лежит **внутри подписанного образа** — подменить
его при включённом Secure Boot нельзя.

> 🔤 **Пароль LUKS придумывайте из латинских букв и цифр.** В ранней загрузке раскладка английская.

---

## Шаг 1. VirtualBox: создать ВМ

> На реальном компьютере пропустите этот шаг и переходите к шагу 2.

- [ ] Скачать ISO: https://archlinux.org/download/
- [ ] Создать ВМ: тип **Linux → Arch Linux (64-bit)**, ОЗУ **4096 МБ**, диск **30 ГБ** (VDI, динамический)
- [ ] **Настройки → Система → Материнская плата**: ✅ **Включить EFI**;
      **Secure Boot — НЕ включать** (почему — см. шаг 8)
- [ ] **Настройки → Носители**: подключить ISO
- [ ] **Настройки → Система → Порядок загрузки**: optical первым
- [ ] Запустить ВМ → выбрать **Arch Linux install medium**

*(Необязательно: после установки `pacman -S virtualbox-guest-utils` + `systemctl enable vboxservice`.)*

---

## Шаг 2. Live-ISO: разметка и шифрование 🔐

- [ ] Проверить UEFI-режим (должно напечатать `64`), интернет и имя диска:

```bash
cat /sys/firmware/efi/fw_platform_size
ping -c 2 archlinux.org
lsblk
```

- [ ] (Необязательно) `loadkeys ru` и `setfont cyr-sun16`
- [ ] Разметить диск:

```bash
cfdisk /dev/sda
```

В `cfdisk`: **gpt** → `New` → `1G` → `Type` → **EFI System** →
`New` → всё оставшееся место → `Type` → **Linux filesystem** → `Write` → `yes` → `Quit`

- [ ] 🔐 Зашифровать второй раздел (пароль вводится дважды; на вопрос ответить `YES`):

```bash
cryptsetup luksFormat --type luks2 /dev/sda2
cryptsetup open /dev/sda2 cryptroot
```

- [ ] Создать файловые системы и смонтировать — **вариант A: ext4**:

```bash
mkfs.fat -F32 /dev/sda1
mkfs.ext4 /dev/mapper/cryptroot
mount /dev/mapper/cryptroot /mnt
mount --mkdir /dev/sda1 /mnt/boot/efi
```

- [ ] **Или вариант B: btrfs** — вместо варианта A:

```bash
mkfs.fat -F32 /dev/sda1
mkfs.btrfs -L archroot /dev/mapper/cryptroot
mount /dev/mapper/cryptroot /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
umount /mnt
mount -o noatime,compress=zstd,subvol=@ /dev/mapper/cryptroot /mnt
mount --mkdir -o noatime,compress=zstd,subvol=@home /dev/mapper/cryptroot /mnt/home
mkdir -p /mnt/boot
mount --mkdir /dev/sda1 /mnt/boot/efi
```

> 🌿 btrfs и LUKS сочетаются нормально. Swap-файл на btrfs не работает — при необходимости zram.

- [ ] Проверить монтирование (должны быть `/mnt` и `/mnt/boot/efi`):

```bash
findmnt --real
```

---

## Шаг 3. Установить систему

- [ ] Установить пакеты (обратите внимание на `cryptsetup`):

```bash
pacstrap -K /mnt base linux-lts linux-firmware intel-ucode amd-ucode cryptsetup \
  systemd-ukify sbctl sudo nano networkmanager
```

- [ ] Записать fstab и войти в chroot:

```bash
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

- [ ] Проверить fstab: корень — `/dev/mapper/cryptroot`, плюс строка для `/boot/efi`
      (в варианте B — ещё и `/home` с `subvol=@home`):

```bash
cat /etc/fstab
```

- [ ] Время, локаль, имя машины, сеть:

```bash
ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime
hwclock --systohc
sed -i 's/#ru_RU.UTF-8/ru_RU.UTF-8/; s/#en_US.UTF-8/en_US.UTF-8/' /etc/locale.gen
locale-gen
echo "LANG=ru_RU.UTF-8" > /etc/locale.conf
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

## Шаг 4. 🔐 Научить initramfs открывать диск

- [ ] Добавить хук `encrypt` (после `block`, до `filesystems`):

```bash
sed -i 's/^HOOKS=.*/HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block encrypt filesystems fsck)/' /etc/mkinitcpio.conf
grep '^HOOKS' /etc/mkinitcpio.conf
```

---

## Шаг 5. Собрать UKI и поставить systemd-boot

- [ ] Создать каталоги и записать параметры ядра.
      **Для ext4 (вариант A):**

```bash
mkdir -p /boot/efi/EFI/Linux /etc/kernel
LUKS_UUID=$(lsblk -no UUID "/dev/$(lsblk -no PKNAME /dev/mapper/cryptroot)")
echo "cryptdevice=UUID=$LUKS_UUID:cryptroot root=/dev/mapper/cryptroot rw" > /etc/kernel/cmdline
cat /etc/kernel/cmdline
```

**Для btrfs (вариант B)** — вместо последней команды `echo`:

```bash
echo "cryptdevice=UUID=$LUKS_UUID:cryptroot root=/dev/mapper/cryptroot rw rootflags=subvol=@" > /etc/kernel/cmdline
```

> В выводе `cat` должен быть настоящий UUID (`cryptdevice=UUID=a1b2c3d4-…:cryptroot …`).
> Если `UUID=` пустое — узнайте его командой `blkid` и впишите строку руками в `nano /etc/kernel/cmdline`.

- [ ] Настроить mkinitcpio на сборку UKI:

```bash
sed -i "s|^PRESETS=.*|PRESETS=('default')|" /etc/mkinitcpio.d/linux-lts.preset
sed -i 's|^default_image=|#default_image=|' /etc/mkinitcpio.d/linux-lts.preset
sed -i 's|^#\?default_uki=.*|default_uki="/boot/efi/EFI/Linux/arch-linux-lts.efi"|' /etc/mkinitcpio.d/linux-lts.preset
sed -i 's|^#\?default_options=.*|default_options="--cmdline /etc/kernel/cmdline"|' /etc/mkinitcpio.d/linux-lts.preset
grep -E "^(PRESETS|#default_image|default_uki|default_options)" /etc/mkinitcpio.d/linux-lts.preset
```

- [ ] Собрать UKI:

```bash
mkinitcpio -P
ls -lh /boot/efi/EFI/Linux/arch-linux-lts.efi
```

- [ ] Установить systemd-boot (ESP не в `/boot`, поэтому путь указываем явно):

```bash
bootctl --esp-path=/boot/efi --boot-path=/boot/efi install
```

- [ ] Настроить меню загрузчика (без этого оно не появится — таймаут по умолчанию `0`,
      и systemd-boot сразу грузит единственный пункт):

```bash
cat > /boot/efi/loader/loader.conf <<'EOF'
timeout 3
default arch-linux-lts
console-mode keep
editor no
EOF
cat /boot/efi/loader/loader.conf
```

> `editor no` обязателен при Secure Boot: иначе в меню можно править параметры ядра.
> В меню systemd-boot входит и сам — удерживайте `Space`/`Tab` сразу после включения.

- [ ] Выйти, отмонтировать, перезагрузиться:

```bash
exit
umount -R /mnt
cryptsetup close cryptroot
reboot
```

> В VirtualBox: перед запуском выключите ВМ и **отключите ISO**.

> 🌱 **Нужно второе ядро как запасной вариант?** В комплекте по умолчанию ставится только `linux-lts`.
> Как добавить обычное `linux` и подписать оба UKI — см. README, раздел «Два ядра: linux + linux-lts».

---

## Шаг 6. Первый запуск

- [ ] В меню systemd-boot выбрать **Arch Linux**
- [ ] Запрос пароля → ввести **пароль LUKS** (символы не отображаются) — один раз
- [ ] Система загрузилась → `user` → `su -`
- [ ] Проверить шифрование и разделы:

```bash
lsblk
findmnt --real
```

Должно быть видно `cryptroot` (корень) и `/dev/sda1` на `/boot/efi`.

---

## Шаг 7. Secure Boot: ключи и подписи

- [ ] Текущее состояние (ожидаемо `Secure Boot: Disabled`):

```bash
sbctl status
```

- [ ] Создать ключи и записать их в прошивку (`-m` сохраняет ключи Microsoft):

```bash
sbctl create-keys
sbctl enroll-keys -m
```

> Если **«Access denied»** — прошивка не в режиме *Setup Mode*: BIOS/UEFI → Secure Boot →
> **Clear / Delete Secure Boot keys** → загрузиться в Arch → повторить.

- [ ] Подписать загрузчик и UKI:

```bash
sbctl sign -s /boot/efi/EFI/systemd/systemd-bootx64.efi
sbctl sign -s /boot/efi/EFI/Linux/arch-linux-lts.efi
```

- [ ] Проверить:

```bash
sbctl verify
sbctl list-files
```

> `sbctl verify` должен пройти **чисто**: на ESP только подписанные файлы, `/boot` — внутри LUKS.

- [ ] Перезагрузиться: `reboot` → пароль LUKS → система грузится

---

## Шаг 8. Включить Secure Boot

### На реальном компьютере

- [ ] Перезагрузка → прошивка (`Del`, `F2`, `F10`) → **Secure Boot → Enabled** → `F10`
- [ ] Ввести пароль LUKS → проверить:

```bash
sbctl status
```

Должно быть `Secure Boot: Enabled`. Готово ✅

### В VirtualBox

- [ ] Ничего включать не нужно — см. предупреждение

> ⚠️ **Почему в VirtualBox Secure Boot не включаем.** VirtualBox хранит только ключи Microsoft
> (`VBoxManage modifynvram … enrollmssignatures`) и не даёт записать свои через интерфейс.
> С включённым SB виртуалка откажется загружать наш подписанный вашим ключом systemd-boot.
> Шаг 7 при этом настоящий: ключи созданы, файлы действительно подписаны.

---

## Шаг 9. Если что-то пошло не так

| Симптом | Что делать |
|---------|------------|
| Снова просит пароль и не пускает | неверный пароль LUKS; раскладка английская, Caps Lock? |
| «wrong UUID» / корень не найден | ошибка в `/etc/kernel/cmdline`. С ISO: `cryptsetup open /dev/sda2 cryptroot` → `mount /dev/mapper/cryptroot /mnt` → `mount /dev/sda1 /mnt/boot/efi` → `arch-chroot /mnt` → `nano /etc/kernel/cmdline` → `mkinitcpio -P` |
| Нет меню, «booting failed» | тот же chroot → `bootctl --esp-path=/boot/efi --boot-path=/boot/efi install` |
| Нет меню systemd-boot, сразу грузится Arch | в `<ESP>/loader/loader.conf` должно быть `timeout 3`; войти в меню принудительно — удерживать `Space`/`Tab` при включении |
| ВМ в **UEFI Interactive Shell** | `FS0:` ↵ → `\EFI\systemd\systemd-bootx64.efi` ↵ |
| «Access denied» при выборе Arch Linux | UKI не подписан: `sbctl sign -s /boot/efi/EFI/Linux/arch-linux-lts.efi` |
| После включения SB чёрный экран | выключить SB в прошивке → `sbctl verify` → переподписать оба файла |
| `sbctl enroll-keys` → «Access denied» | прошивка не в Setup Mode — очистите ключи SB в BIOS |
| Обновили ядро и SB ругается | `sudo mkinitcpio -P` && `sudo sbctl sign-all` |
| Забыли пароль root (LUKS помните) | с ISO откройте и смонтируйте (см. строку 2) → `arch-chroot /mnt` → `passwd` |

---

## После установки (коротко)

- Обновление системы: `sudo pacman -Syu`. UKI пересобирается хуком mkinitcpio и **переподписывается
  автоматически** хуком sbctl. Проверить: `sbctl list-files`, `sbctl verify`.
- 🔐 Вход без пароля через TPM2 (`systemd-cryptenroll --tpm2-device=auto`) — отдельная тема уровня
  «продвинуто»; с UKI сочетается хорошо.
- Swap: только внутри зашифрованного корня (файл `/swapfile`) или zram. **Не** отдельный
  незашифрованный раздел — туда попадает содержимое памяти в открытом виде.
- Дальше: ArchWiki → **dm-crypt**, **Unified kernel image**, **General recommendations**.
