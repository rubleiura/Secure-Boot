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
  строка. Подписывается целиком, поэтому параметры загрузки (`root=`, `rd.luks.name=`) подделать
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
> sudo nano /etc/mkinitcpio.d/linux-lts.preset # default_uki=… и default_options="--cmdline /etc/kernel/cmdline"
> sudo mkinitcpio -P && sudo bootctl install
> sudo bootctl set-timeout 3                    # иначе меню не появится: таймаут по умолчанию 0
> sudo sbctl sign -s /boot/EFI/systemd/systemd-bootx64.efi /boot/EFI/Linux/arch-linux-lts.efi
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

## Secure Boot в VirtualBox — работает

Проверено на реальной установке: VirtualBox позволяет гостю **самому записать ключи** в UEFI-хранилище
виртуальной машины, поэтому в ВМ отрабатывается весь цикл: создание ключей → подпись → включение
Secure Boot → загрузка подписанного UKI. Реальный вывод `sbctl status` / `bootctl status`:

```
Installed:      ✓ sbctl is installed
Setup Mode:     ✓ Disabled
Secure Boot:    ✓ Enabled
Vendor Keys:    microsoft
Firmware: UEFI 2.70 (EDK II 1.00)      ← прошивка VirtualBox
Current Entry: arch-linux-lts.efi      ← загружен UKI
options: root=UUID=… rw rootflags=subvol=@
```

Три особенности VirtualBox, которые **не являются ошибками**:

| Что видно | Почему |
|---|---|
| `No boot loaders listed in EFI Variables`, `Loader: …/EFI/BOOT/BOOTX64.EFI` | ВМ грузит systemd-boot по запасному пути, а не по записи NVRAM. Поэтому `EFI/BOOT/BOOTX64.EFI` **обязательно подписывается** наряду с `EFI/systemd/systemd-bootx64.efi` |
| `TPM2 Support: no`, `Measured UKI: no` | у ВМ нет TPM. Включается при желании: Настройки → Система → Материнская плата → TPM → 2.0 |
| красные строки `vmwgfx … unsupported hypervisor` | VirtualBox показывает Linux-гостю устройство VMware SVGA, ядро грузит драйвер `vmwgfx`. Косметика; лечится переключением графического адаптера на **VBoxSVGA** |

Галочку **Enable Secure Boot** можно ставить сразу при создании ВМ. Если `sbctl enroll-keys -m`
отвечает «Access denied» — снимите её, загрузитесь, выполните шаг с ключами, затем верните обратно.

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

## Тихая загрузка и Plymouth

Настроены **в основном потоке установки, прямо в chroot** — чтобы UKI собирался один раз, уже
с заставкой и без логов. Отдельно ничего доделывать не нужно.

| Что | Где задаётся (в chroot) |
|---|---|
| скрыть логи | `/etc/kernel/cmdline` → `quiet splash loglevel=3` (аналог прежнего `GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"`) |
| заставка | пакет `plymouth` (в pacstrap) + хук `plymouth` в `mkinitcpio.conf` + `Theme=bgrt`, `ShowDelay=0` в `/etc/plymouth/plymouthd.conf` |
| применить | там же, в chroot: `mkinitcpio -P` → после первой загрузки и подписи всё уже работает |

Четыре нюанса:

- `splash` **обязателен** вместе с `quiet` — иначе plymouth включает тему `details` и снова показывает
  служебные сообщения systemd.
- Тема по умолчанию в Arch — **`bgrt`** (не `spinner`): это вариация `spinner`, удерживающая логотип
  прошивки через таблицу BGRT. В VirtualBox BGRT нет, поэтому будет только анимация.
- Список тем: `plymouth-set-default-theme -l` (команды `plymouth-list-themes` не существует).
- Любое изменение `/etc/kernel/cmdline` или `mkinitcpio.conf` требует **пересборки UKI и повторной
  подписи** (`mkinitcpio -P` → `sbctl sign-all`). При включённом Secure Boot редактор параметров
  в меню отключён (`editor no`), поэтому «на лету» строку загрузки не поправить.

Подробности и готовые команды для правок — приложение «Как изменить вид загрузки» в конце каждой инструкции.

## Про хук `microcode` (важно для LUKS-инструкций)

С mkinitcpio 39 микрокод ядра пакуется **внутрь** initramfs одноимённым хуком (флаг `--microcode`
устарел). С UKI это единственный рабочий способ, поэтому в строке HOOKS хук `microcode` обязан
присутствовать:

```
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt filesystems fsck)
```

Если добавляете свои хуки — вставляйте их в существующую строку, а не переписывайте её целиком:
`encrypt` идёт после `block`, `plymouth` — перед `block`.

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

## Контрольные точки при тестировании

Если ставите начисто и хотите быстро понять, где проблема, сверяйтесь с этим списком:

| Когда | Команда | Что должно быть |
|---|---|---|
| после шага «Собрать UKI» | `ls -lh <ESP>/EFI/Linux/arch-linux-lts.efi` | файл существует, ~120–250 МБ |
| после шага «Собрать UKI» | `cat <ESP>/loader/loader.conf` | `timeout 3`, `default arch-linux-lts` |
| после первого запуска | `bootctl status` | «systemd-boot … installed», пункт Arch Linux |
| после первого запуска | `cat /proc/cmdline` | совпадает с `/etc/kernel/cmdline`, в конце `quiet splash loglevel=3` |
| после первого запуска | `plymouth-set-default-theme` | `bgrt`; при загрузке заставка вместо логов |
| после `sbctl sign` | `sbctl list-files` | 4 файла при ESP=`/boot` (systemd-bootx64.efi, BOOTX64.EFI, arch-linux-lts.efi, vmlinuz-linux-lts); 3 файла при ESP=`/boot/efi` (без vmlinuz) |
| после `sbctl sign` | `sbctl verify` | все строки `✓ … is signed`, ошибок нет |
| после включения SB | `sbctl status` | `Secure Boot: Enabled`, `sbctl installed: yes` |
| после включения SB | `bootctl status` | `Secure Boot: enabled (user)`, `Current Entry: arch-linux-lts.efi` |
| для схем 3 и 4 | `lsblk` | виден `cryptroot`, корень — `/dev/mapper/cryptroot` |
| после `pacman -Syu` | `sbctl verify` | подписи на месте (хук sbctl переподписал UKI) |

`<ESP>` — это `/boot` в инструкциях 1 и 3, `/boot/efi` в инструкциях 2 и 4.

## Версия

**v3.4** — тихая загрузка и plymouth перенесены **в основной поток установки (в chroot)**: пакет
`plymouth` добавлен в pacstrap, `quiet splash loglevel=3` — в `/etc/kernel/cmdline`, хук `plymouth`
и `plymouthd.conf` настраиваются до `mkinitcpio -P`, так что UKI собирается один раз.
LUKS-инструкции переведены на systemd-хуки (`systemd`, `sd-vconsole`, **`sd-encrypt`**) и параметр
`rd.luks.name=<UUID>=cryptroot` вместо `cryptdevice=`: с обычным хуком `encrypt` plymouth
проглатывает запрос пароля. Добавлены пояснения к закомментированной строке `#default_image`
в пресете и ожидаемый вывод команды проверки. Приложение сокращено до справочника правок.

**v3.3** — исправлена тема plymouth: по умолчанию в Arch это **`bgrt`**, а не `spinner`
(подтверждается командой `plymouth-set-default-theme`); убрана несуществующая команда
`plymouth-list-themes` — вместо неё `plymouth-set-default-theme -l`; добавлено пояснение,
что в VirtualBox нет BGRT и логотипа прошивки не будет, плюс строка troubleshooting
про чёрный экран (смена темы на `spinner`, адаптер VBoxSVGA).

**v3.2** — добавлено приложение «Тихая загрузка и Plymouth» во все четыре инструкции
(`/etc/kernel/cmdline` + хук `plymouth` + обязательная пересборка и переподпись UKI);
в LUKS-инструкциях восстановлен хук **`microcode`** в строке HOOKS — без него микрокод не попадает
в UKI; добавлены разделы README про тихую загрузку и про `microcode`.

**v3.1** — уточнено по фактическому выводу `sbctl list-files` / `sbctl verify` с тестовой машины:
в схемах с ESP=`/boot` подписывается ещё и `/boot/vmlinuz-linux-lts` (иначе он навсегда остаётся
в отчёте `verify`, а подписанное ядро полезно как запасной вариант загрузки); добавлено примечание
про подписывание сторонних `.efi` на ESP (`memtest86+-efi`, `edk2-shell`).

**v3** — исправлено по результатам тестовой установки: Secure Boot в VirtualBox **работает**
(раньше утверждалось обратное); в список подписываемых файлов добавлен запасной путь
`EFI/BOOT/BOOTX64.EFI` — именно через него VirtualBox грузит systemd-boot; добавлены особенности ВМ
(`vmwgfx`, отсутствие TPM, записи NVRAM) и уточнение про Guest Additions для `linux-lts`.

**v2** — GRUB заменён на **systemd-boot + UKI**: GRUB из Arch при включённом Secure Boot не грузит
модули без shim (`shim_lock_verifier_init: prohibited by secure boot policy`). Ядро по умолчанию —
`linux-lts`. Добавлена настройка `loader.conf` (без неё меню не появляется). Инструкция 4 упрощена
до двух разделов (отдельный незашифрованный `/boot` больше не нужен — ядро упаковано в UKI).
Варианты ext4/btrfs — шаг с выбором A/B.
Предыдущие версии на GRUB лежат в `../_arhiv-grub-GRUB/` — **использовать их нельзя**.

---
*Составлено по ArchWiki (Installation guide, Unified kernel image, systemd-boot, dm-crypt,
Unified Extensible Firmware Interface/Secure Boot) и документации Oracle VirtualBox.
Актуально на октябрь 2026.*
