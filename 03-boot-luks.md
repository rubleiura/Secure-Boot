# Инструкция 3 · ESP в `/boot` · 🔐 LUKS2 (шифрованный корень) · Secure Boot

**Стек:** systemd-boot + UKI + sbctl + LUKS2 · **Разделов:** 2 · **Время:** 45–55 минут
**Где работает:** реальный компьютер (UEFI) и VirtualBox

> ⚠️ Все данные на выбранном диске будут уничтожены.
> ⚠️ **Пароль LUKS восстановить нельзя.** Забыли — данные потеряны. Запишите его в надёжном месте.
> ⚠️ **Почему не GRUB:** при включённом Secure Boot GRUB из Arch отказывается грузить модули без shim
> (`shim_lock_verifier_init: prohibited by secure boot policy` → `grub rescue`). Здесь systemd-boot + UKI.

## Что получится

| Раздел | Размер | Тип / ФС | Точка монтирования | Зашифрован |
|--------|--------|----------|--------------------|------------|
| `/dev/sda1` | 1 ГБ | EFI System / FAT32 | `/boot` | ❌ нет |
| `/dev/sda2` | всё остальное | LUKS2 → ext4 (или btrfs) | `/` (как `cryptroot`) | ✅ да |

ESP **не может быть зашифрован** — прошивка читает загрузчик до ввода пароля.
**Цепочка доверия:** прошивка → подписанный **systemd-boot** → подписанный **UKI** (ядро + initramfs +
параметры внутри одного файла) → initramfs спрашивает пароль LUKS и открывает корень.

Параметр `rd.luks.name=…` зашит **внутри подписанного UKI** — это главное преимущество схемы:
подменить параметры загрузки при включённом Secure Boot нельзя.

> 🔤 **Пароль LUKS придумывайте из латинских букв и цифр.** В ранней загрузке раскладка английская.

---

## Шаг 1. VirtualBox: создать ВМ

> На реальном компьютере пропустите этот шаг и переходите к шагу 2.

- [ ] Скачать ISO: https://archlinux.org/download/
- [ ] Создать ВМ: тип **Linux → Arch Linux (64-bit)**, ОЗУ **4096 МБ**, диск **30 ГБ** (VDI, динамический)
- [ ] **Настройки → Система → Материнская плата**: ✅ **Включить EFI**;
      **Secure Boot — можно включить сразу** (VirtualBox позволяет записать свои ключи
      изнутри гостя; при «Access denied» на шаге записи ключей временно снимите галочку)
- [ ] **Настройки → Носители**: подключить ISO
- [ ] **Настройки → Система → Порядок загрузки**: optical первым
- [ ] Запустить ВМ → выбрать **Arch Linux install medium**

*(Необязательно: после установки `pacman -S virtualbox-guest-utils` + `systemctl enable vboxservice`.
Модули ядра `vboxguest`/`vboxvideo`/`vboxsf` уже входят в пакет `linux-lts` — отдельные
`virtualbox-guest-modules-*` и `virtualbox-guest-dkms` больше не нужны.)*

---

## Шаг 2. Live-ISO: разметка и шифрование 🔐

- [ ] Проверить UEFI-режим (должно напечатать `64`) и интернет:

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
mount --mkdir /dev/sda1 /mnt/boot
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
mount --mkdir /dev/sda1 /mnt/boot
```

> 🌿 btrfs и LUKS сочетаются нормально: шифрование работает ниже, на уровне раздела.
> Swap-файл на btrfs не работает — при необходимости `zram-generator`.

- [ ] Проверить монтирование:

```bash
findmnt --real
```

---

## Шаг 3. Установить систему

- [ ] Установить пакеты (обратите внимание на `cryptsetup`):

```bash
pacstrap -K /mnt base linux-lts linux-firmware intel-ucode amd-ucode cryptsetup \
  systemd-ukify sbctl plymouth sudo nano networkmanager
```

- [ ] Записать fstab и войти в chroot:

```bash
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

- [ ] Проверить fstab: корень — `/dev/mapper/cryptroot`, плюс строка для `/boot`
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

- [ ] Заменить строку HOOKS: systemd-хуки + `sd-encrypt` + `plymouth`:

```bash
sed -i 's/^HOOKS=.*/HOOKS=(base systemd autodetect microcode modconf kms sd-vconsole plymouth block sd-encrypt filesystems fsck)/' /etc/mkinitcpio.conf
grep '^HOOKS' /etc/mkinitcpio.conf
```

- [ ] Задать тему plymouth (`bgrt` — родная тема Arch):

```bash
mkdir -p /etc/plymouth
printf '[Daemon]\nTheme=bgrt\nShowDelay=0\n' > /etc/plymouth/plymouthd.conf
```

> **Почему `sd-encrypt`, а не `encrypt`.** С обычным хуком `encrypt` plymouth **проглатывает запрос
> пароля LUKS**: заставка есть, а ввести пароль негде (это описано в ArchWiki →
> dm-crypt/System configuration). Хук `sd-encrypt` использует `systemd-cryptsetup`, который спрашивает
> пароль через `systemd-ask-password` и рисует поле ввода прямо на заставке.
> Вместе с ним меняются и соседние хуки: `udev` → `systemd`, `keyboard keymap consolefont` → `sd-vconsole`.
>
> Хук `microcode` обязателен: с UKI микрокод ядра упаковывается **внутрь** initramfs (флаг
> `--microcode` устарел). `plymouth` стоит **до** `block` и `sd-encrypt`, чтобы заставка появилась
> раньше запроса пароля. Если в вашей строке HOOKS есть свои хуки — вставляйте их в неё,
> а не переписывайте целиком.

---

## Шаг 5. Собрать UKI и поставить systemd-boot

- [ ] Создать каталоги и записать параметры ядра.
      **Для ext4 (вариант A):**

```bash
mkdir -p /boot/EFI/Linux /etc/kernel
LUKS_UUID=$(lsblk -no UUID "/dev/$(lsblk -no PKNAME /dev/mapper/cryptroot)")
echo "rd.luks.name=$LUKS_UUID=cryptroot root=/dev/mapper/cryptroot rw quiet splash loglevel=3" > /etc/kernel/cmdline
cat /etc/kernel/cmdline
```

**Для btrfs (вариант B)** — вместо последней команды `echo`:

```bash
echo "rd.luks.name=$LUKS_UUID=cryptroot root=/dev/mapper/cryptroot rw rootflags=subvol=@ quiet splash loglevel=3" > /etc/kernel/cmdline
```

> В выводе `cat` должен быть настоящий UUID (`rd.luks.name=a1b2c3d4-…=cryptroot …`),
> а в конце — `quiet splash loglevel=3`: они гасят логи и включают заставку plymouth.
> Если `UUID=` пустое — посмотрите его командой `blkid` и впишите строку руками в `nano /etc/kernel/cmdline`.

- [ ] Настроить mkinitcpio на сборку UKI:

```bash
sed -i "s|^PRESETS=.*|PRESETS=('default')|" /etc/mkinitcpio.d/linux-lts.preset
sed -i 's|^default_image=|#default_image=|' /etc/mkinitcpio.d/linux-lts.preset
sed -i 's|^#\?default_uki=.*|default_uki="/boot/EFI/Linux/arch-linux-lts.efi"|' /etc/mkinitcpio.d/linux-lts.preset
sed -i 's|^#\?default_options=.*|default_options="--cmdline /etc/kernel/cmdline"|' /etc/mkinitcpio.d/linux-lts.preset
grep -E "^(PRESETS|#default_image|default_uki|default_options)" /etc/mkinitcpio.d/linux-lts.preset
```

> Ожидаемый вывод (строка с `#` в начале — так и должно быть):
> ```
> PRESETS=('default')
> #default_image="/boot/initramfs-linux-lts.img"
> default_uki="/boot/EFI/Linux/arch-linux-lts.efi"
> default_options="--cmdline /etc/kernel/cmdline"
> ```
> `#default_image=…` закомментирована **специально**: отдельный initramfs больше не собирается —
> ядро, initramfs (вместе с plymouth) и параметры ядра упаковываются в один UKI.
> `PRESETS=('default')` отключает сборку запасного (fallback) образа, чтобы UKI гарантированно
> помещался на ESP.

- [ ] Собрать UKI и установить systemd-boot:

```bash
mkinitcpio -P
ls -lh /boot/EFI/Linux/arch-linux-lts.efi
bootctl install
```

- [ ] Настроить меню загрузчика (без этого оно не появится — таймаут по умолчанию `0`,
      и systemd-boot сразу грузит единственный пункт):

```bash
cat > /boot/loader/loader.conf <<'EOF'
timeout 3
default arch-linux-lts
console-mode keep
editor no
EOF
cat /boot/loader/loader.conf
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
- [ ] На заставке plymouth появится поле ввода → ввести **пароль LUKS** — один раз
- [ ] Система загрузилась → `user` → `su -`
- [ ] Проверить шифрование:

```bash
lsblk
findmnt /
```

Должно быть видно `cryptroot` и корень на `/dev/mapper/cryptroot`.

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

- [ ] Подписать загрузчик (**оба его экземпляра**) и UKI:

```bash
sbctl sign -s /boot/EFI/systemd/systemd-bootx64.efi
sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI
sbctl sign -s /boot/EFI/Linux/arch-linux-lts.efi
sbctl sign -s /boot/vmlinuz-linux-lts
```

> `bootctl install` кладёт systemd-boot в два места: `<ESP>/EFI/systemd/systemd-bootx64.efi`
> (основное) и `…/EFI/BOOT/BOOTX64.EFI` (запасной путь — по нему грузится, например, VirtualBox).
> Подписывать нужно **оба**, иначе с включённым Secure Boot машина не загрузится.

- [ ] Проверить:

```bash
sbctl verify
sbctl list-files
```

> `/boot/vmlinuz-linux-lts` подписываем не для загрузки UKI, а чтобы отчёт `sbctl verify` был чистым
> (в этой схеме ESP — это и есть `/boot`). Бонус: подписанное ядро можно грузить напрямую как запасной
> вариант (для него придётся раскомментировать `default_image` в пресете).
> Ожидаемый вывод — все строки с галочкой:
> ```
> ✓ /boot/vmlinuz-linux-lts is signed
> ✓ /boot/EFI/BOOT/BOOTX64.EFI is signed
> ✓ /boot/EFI/Linux/arch-linux-lts.efi is signed
> ✓ /boot/EFI/systemd/systemd-bootx64.efi is signed
> ```
> Ставили на ESP что-то ещё (`memtest86+-efi`, `edk2-shell`)? Подпишите и это:
> `sbctl sign -s /boot/memtest86+/memtest.efi`.

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

**Secure Boot в VirtualBox работает по-настоящему**: гость сам записывает ключи в UEFI-хранилище
виртуальной машины, поэтому шаг 7 здесь не учебный, а боевой. Проверено на практике —
`Setup Mode: Disabled`, `Secure Boot: Enabled`, `Vendor Keys: microsoft`, грузится UKI.

- [ ] Выключить ВМ → **Настройки → Система → Материнская плата → ✅ Включить Secure Boot** → запустить
- [ ] В загруженной системе проверить:

```bash
sbctl status      # Setup Mode: Disabled · Secure Boot: Enabled · Vendor Keys: microsoft
bootctl status    # Firmware: UEFI 2.70 (EDK II …) · Secure Boot: enabled (user)
```

> 📌 **Что в VirtualBox выглядит странно, но является нормой:**
> - `No boot loaders listed in EFI Variables` и `Loader: …/EFI/BOOT/BOOTX64.EFI` — ВМ загружает
>   systemd-boot по запасному пути, а не по записи NVRAM. Именно поэтому `EFI/BOOT/BOOTX64.EFI`
>   **обязательно должен быть подписан** (см. шаг 7).
> - `TPM2 Support: no`, `Measured UKI: no` — у ВМ нет TPM. При желании включается:
>   Настройки → Система → Материнская плата → TPM → 2.0 (для этих инструкций не требуется).
> - Красные строки `vmwgfx … unsupported hypervisor` — косметика, см. таблицу ниже.
> - Если `sbctl enroll-keys -m` ответил «Access denied» — снимите галочку Secure Boot,
>   загрузитесь, выполните шаг 7, затем верните галочку.

---

## Шаг 9. Если что-то пошло не так

| Симптом | Что делать |
|---------|------------|
| Снова просит пароль и не пускает | неверный пароль LUKS; раскладка английская, Caps Lock? |
| «wrong UUID» / корень не найден | ошибка в `/etc/kernel/cmdline`. С ISO: `cryptsetup open /dev/sda2 cryptroot` → `mount /dev/mapper/cryptroot /mnt` → `mount /dev/sda1 /mnt/boot` → `arch-chroot /mnt` → исправить `nano /etc/kernel/cmdline` → `mkinitcpio -P` |
| Нет меню, «booting failed» | тот же chroot → `bootctl install` |
| Нет меню systemd-boot, сразу грузится Arch | в `<ESP>/loader/loader.conf` должно быть `timeout 3`; войти в меню принудительно — удерживать `Space`/`Tab` при включении |
| В VirtualBox после включения SB не грузится / «Access denied» | не подписан запасной путь: `sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI` — ВМ грузится именно через него |
| Красные строки `vmwgfx … unsupported hypervisor` при загрузке в ВМ | это не ошибка: VirtualBox показывает Linux-гостю устройство VMware SVGA. Можно игнорировать, либо Настройки → Дисплей → Графический адаптер → **VBoxSVGA** |
| ВМ в **UEFI Interactive Shell** | `FS0:` ↵ → `\EFI\systemd\systemd-bootx64.efi` ↵ |
| «Access denied» при выборе Arch Linux | UKI не подписан: `sbctl sign -s /boot/EFI/Linux/arch-linux-lts.efi` |
| После включения SB чёрный экран | выключить SB в прошивке → `sbctl verify` → переподписать оба файла |
| `sbctl enroll-keys` → «Access denied» | прошивка не в Setup Mode — очистите ключи SB в BIOS |
| Обновили ядро и SB ругается | `sudo mkinitcpio -P` && `sudo sbctl sign-all` |
| Забыли пароль root (LUKS помните) | с ISO откройте и смонтируйте (см. строку 2) → `arch-chroot /mnt` → `passwd` |

---

## После установки (коротко)

- Обновление системы: `sudo pacman -Syu`. UKI пересобирается хуком mkinitcpio и **переподписывается
  автоматически** хуком sbctl. Проверить: `sbctl list-files`, `sbctl verify`.
- 🔐 Вход без пароля через TPM2 (`systemd-cryptenroll --tpm2-device=auto`) — отдельная тема уровня
  «продвинуто»; с UKI она сочетается хорошо, но в чек-лист не входит.
- Swap: только внутри зашифрованного корня (файл `/swapfile`) или zram. **Не** отдельный
  незашифрованный раздел — туда попадает содержимое памяти в открытом виде.
- Тихая загрузка (скрыть логи) и экран Plymouth — см. **приложение в конце файла**.
- Дальше: ArchWiki → **dm-crypt**, **Unified kernel image**, **General recommendations**.

---

## Приложение · Как изменить вид загрузки

Логи и заставка настраиваются **в трёх файлах** и применяются пересборкой UKI:

| Что меняем | Файл |
|---|---|
| параметры ядра (`quiet`, `splash`, `loglevel`, `root=`, `rd.luks.name=`) | `/etc/kernel/cmdline` |
| состав initramfs (хуки `plymouth`, `microcode`, `sd-encrypt`) | `/etc/mkinitcpio.conf` |
| тема и задержка plymouth | `/etc/plymouth/plymouthd.conf` |

После **любого** изменения обязательны пересборка и переподпись — иначе ничего не применится:

```bash
sudo mkinitcpio -P
sudo sbctl sign-all
sudo sbctl verify
sudo reboot
```

> ⚠️ При включённом Secure Boot systemd-boot не даёт править параметры ядра в меню (`editor no`).
> «На лету» строку загрузки не изменить — только пересборкой UKI, при необходимости из chroot с ISO.

### Частые правки

| Задача | Команда |
|---|---|
| Вернуть подробные логи | `sudo sed -i 's/ quiet splash loglevel=3//' /etc/kernel/cmdline` |
| Снова скрыть логи | `sudo sed -i 's/$/ quiet splash loglevel=3/' /etc/kernel/cmdline` |
| Узнать текущую тему | `plymouth-set-default-theme` (в Arch по умолчанию `bgrt`) |
| Список установленных тем | `plymouth-set-default-theme -l` (команды `plymouth-list-themes` не существует) |
| Сменить тему | `sudo plymouth-set-default-theme <имя>` — команда сама перепишет `plymouthd.conf` |
| Убрать plymouth | `sudo sed -i '/^HOOKS=/s/ plymouth block / block /' /etc/mkinitcpio.conf` |
| Запрос пароля LUKS не виден | нажмите `Esc`; проверьте, что в HOOKS стоит `sd-encrypt`, а не `encrypt` (см. шаг 4) |

### Если что-то не так

| Симптом | Что делать |
|---|---|
| Чёрный экран вместо заставки | хук `kms` должен стоять **до** `plymouth`; в VirtualBox нет BGRT — поставьте `Theme=spinner`; попробуйте графический адаптер **VBoxSVGA** вместо VMSVGA |
| Заставка «залипла», система не грузится | нажмите `Esc` — появятся логи; затем верните подробные логи строкой выше и пересоберите UKI |
| После `pacman -Syu` заставка пропала | UKI пересобрался, но не переподписался: `sudo sbctl sign-all && sudo sbctl verify` |
| Логи всё равно видны | в `/etc/kernel/cmdline` должны быть **и** `quiet`, **и** `splash`; после правки — пересборка |
