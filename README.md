# Arch Linux + Secure Boot — 4 чек-листа для новичка

Комплект коротких пошаговых инструкций: установка Arch Linux с **Secure Boot** на реальный компьютер
и отработка той же процедуры в **VirtualBox**.

## Какую инструкцию выбрать

| № | Файл | ESP смонтирован в | Шифрование диска | Разделов |
|---|------|-------------------|------------------|----------|
| 1 | [`01-boot-prostaya.md`](01-boot-prostaya.md) | `/boot` | нет | 2 |
| 2 | [`02-boot-efi-prostaya.md`](02-boot-efi-prostaya.md) | `/boot/efi` | нет | 2 |
| 3 | [`03-boot-luks.md`](03-boot-luks.md) | `/boot` | LUKS2 (весь корень) | 2 |
| 4 | [`04-boot-efi-luks.md`](04-boot-efi-luks.md) | `/boot/efi` | LUKS2 (весь корень) | 2 |

Загрузчик и способ подписи **во всех четырёх одинаковые** — выучить нужно одну последовательность.
Меняются только пути (`/boot` ↔ `/boot/efi`) и три шага с шифрованием (помечены 🔐).

**Не знаете, что выбрать?** Начните с инструкции 1 в VirtualBox — она самая простая.

## Стек: systemd-boot + UKI + sbctl

```
прошивка UEFI ──проверяет подпись──> systemd-boot ──запускает──> UKI
                                                                (ядро + initramfs + параметры
                                                                 в одном подписанном файле)
```

- **UKI (unified kernel image)** — один `.efi`-файл, внутри которого ядро, initramfs и командная
  строка. Подписывается целиком, поэтому параметры загрузки (`root=`, `cryptdevice=`) подделать
  при включённом Secure Boot нельзя. Собирается самим `mkinitcpio`/`ukify`, никаких отдельных
  конфигов загрузчика писать не нужно.
- **systemd-boot** — читает только ESP и сам находит UKI в `EFI/Linux/`. Не умеет читать ext4/btrfs —
  при UKI это и не требуется.
- **sbctl** — создание своих ключей и подпись двух файлов (загрузчик + UKI). Всё из официальных
  репозиториев, без AUR.

### ⚠️ Почему не GRUB

GRUB из репозиториев Arch собран с верификатором `shim_lock`. Если Secure Boot включён, а GRUB
запущен **не** через shim, он отказывается загружать свои модули и ядро:

```
error: kern/efi/sb.c:shim_lock_verifier_init:187: prohibited by secure boot policy
Entering rescue mode...
grub rescue>
```

То есть «просто подписать grubx64.efi через sbctl» **недостаточно**: при включённом SB система
не загрузится. Рабочие варианты с GRUB — только через shim + MOK (`shim-signed` из AUR, синий экран
MokManager с одноразовым паролем), что для новичка хуже. Поэтому в комплекте systemd-boot + UKI.

> **Если вы уже установили GRUB по старой версии инструкции:** выключите Secure Boot в прошивке —
> система загрузится. Затем в работающей системе выполните переход (5 команд):
> ```bash
> sudo pacman -S systemd-ukify
> sudo mkdir -p /etc/kernel && echo "root=UUID=$(findmnt -no UUID /) rw" | sudo tee /etc/kernel/cmdline
> sudo nano /etc/mkinitcpio.d/linux.preset     # default_uki=… и default_options="--cmdline /etc/kernel/cmdline"
> sudo mkinitcpio -P && sudo bootctl install
> sudo sbctl sign -s /boot/EFI/systemd/systemd-bootx64.efi /boot/EFI/Linux/arch-linux.efi
> ```
> Точные шаги — в шаге 4 нужной вам инструкции. После этого GRUB можно удалить: `sudo pacman -Rns grub`.

## Два ядра: linux + linux-lts

Во всех четырёх инструкциях по умолчанию ставится **`linux-lts`** — ядро с длительной поддержкой:
обновляется реже, ведёт себя предсказуемее. Обычное `linux` не устанавливается, поэтому UKI собирается
один и в меню systemd-boot ровно один пункт «Arch Linux».

**Нужны оба ядра** (обычное + LTS как запасное, если свежее не загрузится)? Выполните в установленной
системе:

```bash
sudo pacman -S linux
ESP=/boot            # в инструкциях 2 и 4 замените на: ESP=/boot/efi
for p in linux linux-lts; do
  sudo sed -i "s|^PRESETS=.*|PRESETS=('default')|" /etc/mkinitcpio.d/$p.preset
  sudo sed -i 's|^default_image=|#default_image=|' /etc/mkinitcpio.d/$p.preset
  sudo sed -i "s|^#\?default_uki=.*|default_uki=\"$ESP/EFI/Linux/arch-$p.efi\"|" /etc/mkinitcpio.d/$p.preset
  sudo sed -i "s|^#\?default_options=.*|default_options=\"--cmdline /etc/kernel/cmdline\"|" /etc/mkinitcpio.d/$p.preset
done
sudo mkinitcpio -P
sudo sbctl sign -s $ESP/EFI/Linux/arch-linux.efi
sudo sbctl list-files
```

- Оба UKI помещаются на ESP размером 1 ГБ (каждый образ ~120–200 МБ).
- В меню появятся два пункта с одинаковым названием «Arch Linux»: они различаются версией ядра и
  идентификатором (`arch-linux` / `arch-linux-lts`) — их видно в строке состояния внизу экрана,
  когда пункт выбран.
- Чтобы LTS грузился по умолчанию, добавьте в `$ESP/loader/loader.conf` строку `default arch-linux-lts`.
- После обновления ядер UKI пересобираются сами, подписи восстанавливает хук sbctl. Если что-то
  не подписалось: `sudo sbctl sign-all`.

---

## Важно про VirtualBox и Secure Boot

> ⚠️ VirtualBox хранит **только ключи Microsoft** (`VBoxManage modifynvram … enrollmssignatures`)
> и не даёт записать свои ключи через интерфейс. Свои ключи — это как раз то, что создаёт `sbctl`.
>
> Поэтому в виртуальной машине Secure Boot **остаётся выключенным**: вы полностью и по-настоящему
> отрабатываете создание ключей и подпись загрузчика с UKI, но прошивка ВМ их не примет.
> На реальном компьютере Secure Boot включается и работает по-настоящему — раздел
> «Включить Secure Boot» (шаг 7 в инструкциях 1–2, шаг 8 в инструкциях 3–4).

## ext4 или btrfs?

В каждой инструкции шаг «Создать файловые системы» дан в двух вариантах — **A (ext4)** и **B (btrfs)**.
Выбираете один, остальная инструкция не меняется. Выполнять оба нельзя — второй перезапишет первый.

| | ext4 (вариант A) | btrfs (вариант B) |
|---|---|---|
| Сложность | минимальная | +4 команды (подтома `@`, `@home`) |
| Сжатие | нет | zstd, экономит место |
| Снапшоты | нет | да (`snapper` / `timeshift`) |
| Swap-файл | работает | **не работает** → нужен zram |

Отличия в командах только два: `rootflags=subvol=@` в `/etc/kernel/cmdline` и строка `subvol=@home`
в fstab (её `genfstab` добавляет сам). Благодаря UKI загрузчик не читает `/boot` вообще, поэтому
сжатие btrfs безопасно — никаких `chattr +C` не требуется.

## Что НЕ покрыто (осознанно, ради краткости)

- swap / zram, LVM, отдельный `/home` (кроме btrfs-варианта)
- снапшоты btrfs (`snapper`, `timeshift`) и откат системы
- вторая ОС (dual-boot с Windows)
- TPM2-авторазблокировка LUKS (`systemd-cryptenroll`)
- GRUB, shim и MOK — не используются намеренно (см. выше)
- графическое окружение, звук, драйверы — ArchWiki «General recommendations»

## Правила работы с чек-листом

1. Читайте шаг → выполняйте команду → ставьте `x` в квадратике.
2. Все команды копируются целиком, по одной. Блоки копируются полностью, включая `sed` и `echo`.
3. Замените на свои: `user` (имя пользователя), `arch-pc` (имя машины), `Europe/Moscow` (часовой пояс).
   UUID разделов подставляются командами автоматически — переписывать их руками не нужно.
4. Имя диска в примерах — `/dev/sda` (VirtualBox). На ПК это обычно `/dev/nvme0n1`:
   проверьте командой `lsblk` и замените **во всех** командах.
5. После каждого блока с `sed`/`echo` в инструкции есть команда проверки (`cat`, `grep`, `ls`) —
   не пропускайте её: она показывает, что получилось.

---
*Составлено по ArchWiki (Installation guide, Unified kernel image, systemd-boot, dm-crypt,
Unified Extensible Firmware Interface/Secure Boot) и документации Oracle VirtualBox.
Актуально на октябрь 2026.*
