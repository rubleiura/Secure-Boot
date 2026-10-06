# Инструкция 1 · ESP в `/boot` · простая разметка · Secure Boot

**Стек:** systemd-boot + UKI + sbctl · **Разделов:** 2 · **Время:** 35–45 минут
**Где работает:** реальный компьютер (UEFI) и VirtualBox

> ⚠️ **Почему не GRUB.** При включённом Secure Boot GRUB из репозиториев Arch включает внутренний
> верификатор `shim_lock` и отказывается загружать свои модули и ядро, если его запустил не shim:
> `error: kern/efi/sb.c:shim_lock_verifier_init:187: prohibited by secure boot policy` → `grub rescue`.
> Подписанный sbctl GRUB **не загружается** при включённом SB. Поэтому здесь systemd-boot + UKI.

## Что получится

| Раздел | Размер | Тип / ФС | Точка монтирования |
|--------|--------|----------|--------------------|
| `/dev/sda1` | 1 ГБ | EFI System / FAT32 | `/boot` |
| `/dev/sda2` | всё остальное | ext4 (или btrfs) | `/` |

**Цепочка доверия:** прошивка проверяет подпись **systemd-boot** → systemd-boot запускает **UKI**
(один файл: ядро + initramfs + параметры командной строки) → прошивка проверяет подпись **UKI**.
Оба файла подписаны вашим ключом через `sbctl`. Параметры ядра лежат внутри подписанного образа —
подменить их при включённом Secure Boot невозможно.

---

## Шаг 1. VirtualBox: создать ВМ

> На реальном компьютере пропустите этот шаг и переходите к шагу 2.

- [ ] Скачать ISO: https://archlinux.org/download/
- [ ] Создать ВМ: тип **Linux → Arch Linux (64-bit)**, ОЗУ **4096 МБ**, диск **30 ГБ** (VDI, динамический)
- [ ] **Настройки → Система → Материнская плата**: ✅ **Включить EFI**;
      **Secure Boot — можно включить сразу** (VirtualBox позволяет записать свои ключи
      изнутри гостя; при «Access denied» на шаге записи ключей временно снимите галочку)
- [ ] **Настройки → Носители**: подключить скачанный ISO к виртуальному приводу
- [ ] **Настройки → Система → Порядок загрузки**: optical первым
- [ ] Запустить ВМ → в меню ISO выбрать **Arch Linux install medium**

*(Необязательно, для нормального размера экрана в ВМ после установки:
`pacman -S virtualbox-guest-utils` и `systemctl enable vboxservice`.
Модули ядра `vboxguest`/`vboxvideo`/`vboxsf` уже входят в пакет `linux-lts` — отдельные
`virtualbox-guest-modules-*` и `virtualbox-guest-dkms` больше не нужны и не ставятся.)*

---

## Шаг 2. Live-ISO: разметить диск

- [ ] Проверить, что загрузка в UEFI-режиме (должно напечатать `64`):

```bash
cat /sys/firmware/efi/fw_platform_size
```

- [ ] (Необязательно) русская раскладка и шрифт в консоли:

```bash
loadkeys ru
setfont cyr-sun16
```

- [ ] Проверить интернет и найти свой диск:

```bash
ping -c 2 archlinux.org
lsblk
```

- [ ] Разметить диск:

```bash
cfdisk /dev/sda
```

В `cfdisk`: **gpt** → `New` → `1G` → `Type` → **EFI System** →
`New` → всё оставшееся место → `Type` → **Linux filesystem** → `Write` → `yes` → `Quit`

- [ ] Создать файловые системы и смонтировать — **вариант A: ext4** (проще):

```bash
mkfs.fat -F32 /dev/sda1
mkfs.ext4 /dev/sda2
mount /dev/sda2 /mnt
mount --mkdir /dev/sda1 /mnt/boot
```

- [ ] **Или вариант B: btrfs** (сжатие zstd, снапшоты) — вместо варианта A:

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

> 🌿 Выполнять только один вариант: A или B. `genfstab` сам пропишет `subvol=` и `compress=zstd`.
> Swap-файл на btrfs не работает — при необходимости используйте `zram-generator`.

- [ ] Проверить монтирование:

```bash
findmnt --real
```

---

## Шаг 3. Установить систему

- [ ] Установить базовый набор пакетов:

```bash
pacstrap -K /mnt base linux-lts linux-firmware intel-ucode amd-ucode \
  systemd-ukify sbctl sudo nano networkmanager
```

- [ ] Записать fstab и войти в новую систему:

```bash
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

- [ ] Время, локаль, имя машины (часовой пояс замените на свой):

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

## Шаг 4. Собрать UKI и поставить systemd-boot

- [ ] Создать каталог для UKI и записать параметры ядра.
      **Для ext4 (вариант A):**

```bash
mkdir -p /boot/EFI/Linux /etc/kernel
echo "root=UUID=$(findmnt -no UUID /) rw" > /etc/kernel/cmdline
cat /etc/kernel/cmdline
```

**Для btrfs (вариант B)** — вместо команды `echo` выше:

```bash
echo "root=UUID=$(findmnt -no UUID /) rw rootflags=subvol=@" > /etc/kernel/cmdline
```

> В выводе `cat` должен быть реальный UUID (длинная строка вида `a1b2c3d4-…`), а не пустое место.

- [ ] Настроить mkinitcpio на сборку UKI:

```bash
sed -i "s|^PRESETS=.*|PRESETS=('default')|" /etc/mkinitcpio.d/linux-lts.preset
sed -i 's|^default_image=|#default_image=|' /etc/mkinitcpio.d/linux-lts.preset
sed -i 's|^#\?default_uki=.*|default_uki="/boot/EFI/Linux/arch-linux-lts.efi"|' /etc/mkinitcpio.d/linux-lts.preset
sed -i 's|^#\?default_options=.*|default_options="--cmdline /etc/kernel/cmdline"|' /etc/mkinitcpio.d/linux-lts.preset
grep -E "^(PRESETS|#default_image|default_uki|default_options)" /etc/mkinitcpio.d/linux-lts.preset
```

- [ ] Собрать UKI и убедиться, что файл создан:

```bash
mkinitcpio -P
ls -lh /boot/EFI/Linux/arch-linux-lts.efi
```

> Запасной (fallback) образ мы отключили строкой `PRESETS=('default')` — так UKI один и он
> гарантированно помещается на ESP.


- [ ] Установить systemd-boot:

```bash
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

> 🌱 **Нужно второе ядро как запасной вариант?** В комплекте по умолчанию ставится только `linux-lts`.
> Как добавить обычное `linux` и подписать оба UKI — см. README, раздел «Два ядра: linux + linux-lts».

---

## Шаг 5. Первый запуск

- [ ] Выйти, отмонтировать, перезагрузиться:

```bash
exit
umount -R /mnt
reboot
```

> В VirtualBox: перед запуском выключите ВМ и **отключите ISO** в Настройки → Носители.

- [ ] Появится меню systemd-boot → выбрать **Arch Linux** → система загрузилась
- [ ] Войти как `user` → `su -` (или `sudo -i`)
- [ ] Проверить сеть: `ping -c 2 archlinux.org`

---

## Шаг 6. Secure Boot: ключи и подписи

- [ ] Текущее состояние (ожидаемо `Secure Boot: Disabled`):

```bash
sbctl status
```

- [ ] Создать свои ключи и записать их в прошивку (`-m` сохраняет ключи Microsoft):

```bash
sbctl create-keys
sbctl enroll-keys -m
```

> Если **«Access denied»** — прошивка не в режиме *Setup Mode*: зайдите в BIOS/UEFI → Secure Boot →
> **Clear / Delete Secure Boot keys** (или «Reset to Setup Mode») → загрузитесь в Arch → повторите.

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

> `/boot/vmlinuz-linux-lts` подписываем не потому, что он нужен для загрузки UKI, а потому что в этой
> схеме ESP — это и есть `/boot`, и неподписанный файл навсегда останется в отчёте `sbctl verify`.
> Заодно подписанное ядро понадобится, если позже вы вернёте обычную запись загрузки (Type #1)
> с отдельным initramfs — для неё нужно раскомментировать `default_image` в пресете.
>
> Ожидаемый вывод `sbctl verify` — все строки с галочкой:
> ```
> ✓ /boot/vmlinuz-linux-lts is signed
> ✓ /boot/EFI/BOOT/BOOTX64.EFI is signed
> ✓ /boot/EFI/Linux/arch-linux-lts.efi is signed
> ✓ /boot/EFI/systemd/systemd-bootx64.efi is signed
> ```
> Ставили на ESP что-то ещё (`memtest86+-efi`, `edk2-shell`)? Подпишите и это:
> `sbctl sign -s /boot/memtest86+/memtest.efi`.

- [ ] Перезагрузиться: `reboot` → система грузится как обычно

---

## Шаг 7. Включить Secure Boot

### На реальном компьютере

- [ ] Перезагрузка → войти в прошивку (`Del`, `F2`, `F10`) → **Secure Boot → Enabled** → `F10`
- [ ] Система загрузилась → проверить:

```bash
sbctl status
```

Должно быть `Secure Boot: Enabled`. Готово ✅

### В VirtualBox

**Secure Boot в VirtualBox работает по-настоящему**: гость сам записывает ключи в UEFI-хранилище
виртуальной машины, поэтому шаг 6 здесь не учебный, а боевой. Проверено на практике —
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
>   **обязательно должен быть подписан** (см. шаг 6).
> - `TPM2 Support: no`, `Measured UKI: no` — у ВМ нет TPM. При желании включается:
>   Настройки → Система → Материнская плата → TPM → 2.0 (для этих инструкций не требуется).
> - Красные строки `vmwgfx … unsupported hypervisor` — косметика, см. таблицу ниже.
> - Если `sbctl enroll-keys -m` ответил «Access denied» — снимите галочку Secure Boot,
>   загрузитесь, выполните шаг 6, затем верните галочку.

---

## Шаг 8. Если что-то пошло не так

| Симптом | Что делать |
|---------|------------|
| Нет меню, сразу «booting failed» | в live-ISO: `mount /dev/sda2 /mnt` → `mount /dev/sda1 /mnt/boot` → `arch-chroot /mnt` → `bootctl install` |
| Нет меню systemd-boot, сразу грузится Arch | в `<ESP>/loader/loader.conf` должно быть `timeout 3`; войти в меню принудительно — удерживать `Space`/`Tab` при включении |
| В VirtualBox после включения SB не грузится / «Access denied» | не подписан запасной путь: `sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI` — ВМ грузится именно через него |
| Красные строки `vmwgfx … unsupported hypervisor` при загрузке в ВМ | это не ошибка: VirtualBox показывает Linux-гостю устройство VMware SVGA. Можно игнорировать, либо Настройки → Дисплей → Графический адаптер → **VBoxSVGA** |
| ВМ в **UEFI Interactive Shell** | наберите `FS0:` ↵, затем `\EFI\systemd\systemd-bootx64.efi` ↵ |
| Меню есть, но «Access denied» при выборе Arch Linux | файл не подписан: `sbctl verify` → `sbctl sign -s /boot/EFI/Linux/arch-linux-lts.efi` |
| После включения SB чёрный экран | выключите SB в прошивке, загрузитесь, `sbctl verify`, переподпишите оба файла |
| `sbctl enroll-keys` → «Access denied» | прошивка не в Setup Mode — очистите ключи SB в BIOS |
| Система грузится, но «wrong UUID» / не нашла корень | ошибка в `/etc/kernel/cmdline` → `nano /etc/kernel/cmdline`, исправить, затем `mkinitcpio -P` и `sbctl sign -s /boot/EFI/Linux/arch-linux-lts.efi` |
| Обновили ядро и SB ругается | `sudo mkinitcpio -P` && `sudo sbctl sign-all` |
| Забыли пароль root | с ISO: `mount /dev/sda2 /mnt` → `mount /dev/sda1 /mnt/boot` → `arch-chroot /mnt` → `passwd` |

---

## После установки (коротко)

- Обновление системы: `sudo pacman -Syu`. UKI пересобирается хуком mkinitcpio и **переподписывается
  автоматически** хуком sbctl. Проверить: `sbctl list-files`, `sbctl verify`.
- Тихая загрузка (скрыть логи) и экран Plymouth — см. **приложение в конце файла**.
- Дальше: ArchWiki → **General recommendations** (пользователи, графика, звук).

---

## Приложение · Тихая загрузка и Plymouth (необязательно)

Делайте это **после** того, как убедились, что система стабильно загружается с включённым Secure Boot.

> ⚠️ При включённом Secure Boot systemd-boot **не даёт править параметры ядра** прямо в меню
> (у нас задано `editor no`). С «тихой» загрузкой причину сбоя не увидеть — для отладки придётся
> убирать `quiet` и пересобирать UKI из chroot.

Раньше эти настройки жили в `/etc/default/grub` (`GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"`).
С UKI параметры ядра берутся из **`/etc/kernel/cmdline`** — править нужно только этот файл.

### 1. Скрыть логи загрузки

- [ ] Дописать параметры в конец строки:

```bash
sed -i 's/$/ quiet splash loglevel=3/' /etc/kernel/cmdline
cat /etc/kernel/cmdline
```

> `splash` обязателен для plymouth: без него включается тема `details`, которая снова выводит
> служебные сообщения systemd.

### 2. Plymouth — экран загрузки с анимацией

- [ ] Установить plymouth:

```bash
pacman -S plymouth
```

- [ ] Посмотреть текущую тему и список установленных:

```bash
plymouth-set-default-theme        # в Arch по умолчанию — bgrt
plymouth-set-default-theme -l     # все установленные темы (или: ls /usr/share/plymouth/themes)
```

> 🎨 **Тема по умолчанию в Arch — `bgrt`, а не `spinner`.** Это вариация `spinner`, которая
> дополнительно удерживает на экране логотип прошивки, если доступна таблица BGRT
> (Boot Graphics Resource Table). На реальном компьютере будет логотип производителя + анимация.
> **В VirtualBox BGRT отсутствует**, поэтому останется только анимация — это норма, а не поломка.

- [ ] Добавить хук `plymouth` в initramfs.

```bash
sed -i '/^HOOKS=/s/ block / plymouth block /' /etc/mkinitcpio.conf
grep '^HOOKS' /etc/mkinitcpio.conf
```

- [ ] (Необязательно) убрать задержку перед показом заставки, **сохранив родную тему**:

```bash
mkdir -p /etc/plymouth
printf '[Daemon]\nTheme=bgrt\nShowDelay=0\n' > /etc/plymouth/plymouthd.conf
```

> Нужна другая тема — замените `bgrt` на любую из `plymouth-set-default-theme -l`
> или выполните `plymouth-set-default-theme <имя>`: команда сама перезапишет этот файл.

### 3. Пересобрать UKI и переподписать

- [ ] Без этого шага изменения не попадут в загрузчик:

```bash
mkinitcpio -P
sbctl sign-all
sbctl verify
```

- [ ] Перезагрузиться: `reboot` → вместо логов появится заставка

### Если что-то не так

| Симптом | Что делать |
|---|---|
| Чёрный экран вместо заставки | нужен KMS: хук `kms` должен стоять **до** `plymouth`. В VirtualBox теме `bgrt` нечего показывать (нет BGRT) — поставьте `Theme=spinner` в `/etc/plymouth/plymouthd.conf`. Также попробуйте графический адаптер **VBoxSVGA** вместо VMSVGA |
| Заставка «залипла», система не грузится | нажмите `Esc` — появятся логи; затем уберите `quiet splash` из `/etc/kernel/cmdline` → `mkinitcpio -P` → `sbctl sign-all` |
| После `pacman -Syu` заставка пропала | UKI пересобрался, но не переподписался: `sudo sbctl sign-all && sudo sbctl verify` |
| Вернуть подробные логи | `sed -i 's/ quiet splash loglevel=3//' /etc/kernel/cmdline` → `mkinitcpio -P` → `sbctl sign-all` |
