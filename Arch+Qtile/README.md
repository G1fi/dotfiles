# Галерея

TODO

# Информция

TODO

# Установка Arch Linux

- Для двойной загрузки с Windows руководствуемся разделом на [Wiki](https://wiki.archlinux.org/title/Dual_boot_with_Windows)
- Важно уделить внимание функции Secure Boot — раздел на [Wiki](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot)

## Подготовка

Создайте `EFI system partition` необходимого размера

```bash
lsblk
fdisk /dev/диск_для_разметки
```

И создайте файловую систему на разделе

```bash
mkfs.fat -F32 /dev/системный_раздел_efi
```

Проведите стандартный процесс установки Windows, отключите Fast Startup и настройте ОС на использование UTC

```
reg add "HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /d 1 /t REG_DWORD /f
```

## Процесс установки

Весь процесс установки описан в мануале на [Wiki](https://wiki.archlinux.org/title/Installation_guide), дальнейшие действия руководствуются именно им

### Для удобства можно открыть гайды

Переключение между TTY `(Alt + F1-12)`

```bash
Installation_guide
lynx https://github.com/G1fi/dotfiles/tree/main/Arch%2BQtile
```

### Подключение к сети

Ethernet по кабелю должен заработать сразу

Подключение к WI-FI с помощью утилиты `iwctl`

Проверка подключения — `ping archlinux.org`

### Разметка дисков

Пример схемы UEFI с GPT

| Точка монтирования | Раздел                    | Тип раздела           | Рекомендуемый размер |
| :----------------- | :------------------------ | :-------------------- | :------------------- |
| /mnt               | /dev/корневой_раздел      | Linux x86-64 root (/) | Остаток              |
| /mnt/efi           | /dev/системный_раздел_efi | Cистемный раздел EFI  | Минимум 300 МиБ      |
| [SWAP]             | /dev/раздел_подкачки      | Linux swap            | Более 512 МиБ        |

Руководство по разметке — [Wiki](https://wiki.archlinux.org/title/Partitioning)

Если на диске уже есть системный раздел EFI, то следует использовать существующий

Просмотрите список накопителей

```bash
lsblk
```

И измените таблицу разделов на нужном накопителе

```bash
fdisk /dev/диск_для_разметки
```

### Форматирование разделов

Отформатируйте созданные разделы в подходящие файловые системы.

```bash
mkfs.btrfs /dev/корневой_раздел
mkfs.fat -F32 /dev/системный_раздел_efi
mkswap /dev/раздел_подкачки
```

### Монтирование разделов

```bash
mount /dev/корневой_раздел /mnt
mount --mkdir /dev/системный_раздел_efi /mnt/efi
swapon /dev/раздел_подкачки
```

### Установка основных пакетов

При необходимости замени `amd-ucode` на `intel-ucode`

```bash
pacstrap -K /mnt base base-devel linux-zen linux-zen-headers linux-firmware amd-ucode grub efibootmgr os-prober networkmanager sudo nano git
```

### Настройка системы

#### Генерация `fstab` и `chroot` в систему

```bash
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

#### Настройка времени

```bash
ln -sf /usr/share/zoneinfo/Регион/Город /etc/localtime
hwclock --systohc
date
```

#### Настройка локали

Раскоментируйте нужные строки

```bash
nano /etc/locale.gen
```

```bash
en_US.UTF-8
ru_RU.UTF-8
```

Сгенерируйте локали

```bash
locale-gen
```

Установите системную локаль

```bash
echo LANG=ru_RU.UTF-8 > /etc/locale.conf
```

И измените шрифт консоли

```bash
echo FONT=cyr-sun16 > /etc/vconsole.conf
```

#### Настройка сети

Задайте свой `HostName`

```bash
echo HostName > /etc/hostname
```

Отредактируйте `hosts`

```bash
nano /etc/hosts
```

```
127.0.0.1        localhost
::1              localhost
127.0.1.1        HostName.localdomain        HostName
```

Включите необходимые сервисы

```bash
systemctl enable NetworkManager.service
```

#### Настройка менеджера пакетов

Присвойте опции `ParallelDownloads` положительное значение, равное желаемому количеству одновременно загружаемых пакетов

```bash
nano /etc/pacman.conf
```

```
UseSyslog
Color
CheckSpace
VerbosePkgLists
ParallelDownloads=5
ILoveCandy
```

Остальные настройки просто делают вывод менеджера более приятным глазу

#### Настройка загрузчика grub

Установка

```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB --removable
```

Раскоментируйте параметр для обнаружения других ОС

```bash
nano /etc/default/grub
```

```
GRUB_DISABLE_OS_PROBER=false
```

Генерация конфига

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

#### Настройка пользователей

Установка пароля `root`, создание пользователя `username`

```bash
passwd
useradd -m -g users -G wheel -s /bin/bash username
passwd username
```

Выдача прав `sudo`, раскоментируйте строку

```bash
EDITOR=nano visudo
```

```
%wheel      ALL=(ALL:ALL) ALL
```

### Завершение и перезагрузка

```bash
exit
umount -R /mnt
reboot now
```

# Настройка YAY

#### Установка

```bash
sudo pacman -S --needed git base-devel  
git clone https://aur.archlinux.org/yay.git  
cd yay  
makepkg -si
```

#### Первое использование

Обновление пакетов разработки

- Используйте `yay -Y --gendb` для создания базы данных пакетов разработки для *-git пакетов, которые были установлены без yay. Эта команда должна быть выполнена только один раз.
- После этого `yay -Syu --devel` будет проверять обновления пакетов разработки.
- Используйте `yay -Y --devel --save`, чтобы сделать обновления пакетов разработки постоянно включенными (тогда `yay` и `yay -Syu` будут всегда проверять пакеты разработки).

# Установка Qtile

```bash
pacman -S xorg-server qtile
```

Копирование дефолтного конфига

```bash
mkdir -p ~/.config/qtile/  
cp /usr/share/doc/qtile_dir/default_config.py ~/.config/qtile/config.py
```

Отредактируйте xinitrc
```bash
nano ~/.xinitrc
```

```
exec qtile
```

Запуск командой `startx`


# TODO

- Настройка оконного менеджера
- Доп пакеты
  nvidia-dkms nvidia-settings alacritty git fish alsa dosfstools mtools ntfs-3g btrfs-progs virt-manager
- Снапшоты
