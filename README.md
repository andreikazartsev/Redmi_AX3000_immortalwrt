ImmortalWrt For Redmi AX3000
============================

Known issue:
------------

- Same as [openwrt-redmi-ax3000](https://github.com/hzyitc/openwrt-redmi-ax3000)

Default login address: http://192.168.1.1 or http://immortalwrt.lan, username: __root__, password: _none_.

How to build (Ubuntu 24)
============
### Requirements
```bash
sudo bash -c 'bash <(curl -s https://build-scripts.immortalwrt.org/init_build_environment.sh)'
```

### Quickstart
```bash
# Clone this repository
git clone https://github.com/kmiit/Redmi_AX3000_immortalwrt
cd Redmi_AX3000_immortalwrt

# Update and install feeds
./scripts/feeds update -a
./scripts/feeds install -a

# Configure for your device
make menuconfig
# Or check CI scrip for basic config.

# Build
make -j$(nproc)
```

How To install
==============
> Please refer to https://github.com/hzyitc/openwrt-redmi-ax3000?tab=readme-ov-file#how-to-install


## Related Repositories
- [openwrt-redmi-ax3000](https://github.com/hzyitc/openwrt-redmi-ax3000)
- [ImmortalWrt](https://github.com/immortalwrt/immortalwrt)
- [LuCI Web Interface](https://github.com/immortalwrt/luci): Modern and modular interface to control the device via a web browser.
- [ImmortalWrt Packages](https://github.com/immortalwrt/packages): Community repository of ported packages.
- [OpenWrt Routing](https://github.com/openwrt/routing): Packages specifically focused on (mesh) routing.
- [OpenWrt Video](https://github.com/openwrt/video): Packages specifically focused on display servers and clients (Xorg and Wayland).
- Версия подготовлена для использования Podkop и Zapret, добавлены не достающие пакеты (sing-box-tiny, jq, dnsmasq-full, kmod-nft-tproxy, и т.д), для установки podkop нужно форматировать раздел mtd20 и объединить его в общий overlay. Если нет никакого понимания как это сделать то можно обратиться к ИИ и под его руководством сделать необходимые шаги для расширения памяти.
- в последнем релизе в разделе software память должна быть около 5 мб, если больше или меньше, то вы установили не тот релиз.
- Для установки используем обычный раздел openwrt обновление/восстановление в роутере и используем файл с расширением ubi.

Пошаговая инструкция для расширения памяти, и установки podkop, все действия выполняем через ssh подключение(как вариант putty):
!!!Инструкция может быть не актуальной на момент ее использования, все действия прописанные ниже никак не окирпичат ваше устройство, так как основная система лежит в другом разделе!!!

Шаг 1: Полная очистка mtd20 
# Отключаем ubi1, если он ещё подключён
ubidetach -d 1 2>/dev/null

# Стираем весь раздел mtd20 (важно!)
ubiformat /dev/mtd20 -y

Должно появиться сообщение вроде:
ubiformat: mtd20 (nand), size 32505856 bytes (31.0 MiB), 248 eraseblocks
ubiformat: formatting eraseblock 247 -- 100 % complete

Шаг 2: Форматирование в ext4
# Форматируем в ext4
mkfs.ext4 -F /dev/mtdblock20

# Создаём точку монтирования для проверки
mkdir -p /mnt/ext_overlay

# Монтируем и проверяем
mount /dev/mtdblock20 /mnt/ext_overlay
df -h /mnt/ext_overlay

Ожидаемый результат примерно такой:
Filesystem                Size      Used Available Use%
/dev/mtdblock20          27.3M     46.0K     25.1M   0%

Шаг 3: Копирование текущего overlay
# Копируем текущие настройки на новый раздел
cp -a /overlay/* /mnt/ext_overlay/

# Проверяем, что файлы скопировались
ls -la /mnt/ext_overlay

# Отмонтируем
umount /mnt/ext_overlay

Шаг 4: Создаем fstab 
cat > /etc/config/fstab << 'EOF'
config 'global'
    option anon_swap '0'
    option anon_mount '0'
    option auto_swap '1'
    option auto_mount '1'
    option delay_root '5'
    option check_fs '0'

config 'mount'
    option target '/overlay'
    option device '/dev/mtdblock20'
    option fstype 'ext4'
    option enabled '1'
    option is_rootfs '1'
EOF

Шаг 5: Создаем preinit-скрипт с загрузкой драйверов И созданием устройства

cat > /etc/preinit.d/10_ext4_overlay << 'EOF'
#!/bin/sh

# Загружаем модули ext4 и зависимости
modprobe mbcache 2>/dev/null
modprobe jbd2 2>/dev/null
modprobe ext4 2>/dev/null

# Загружаем mtdblock driver
modprobe mtdblock 2>/dev/null

# Даем время на создание устройств
sleep 1
EOF
chmod +x /etc/preinit.d/10_ext4_overlay

Шаг 6: Проверяем конфигурацию

cat /etc/config/fstab
ls -l /etc/preinit.d/

Шаг 7: Перезагрузка и проверка
reboot
После перезагрузки:
df -h /overlay
mount | grep overlay

Если всё заработало, вы увидите ~27 МБ и файловую систему ext4.

Затем установите Podkop:
cd /tmp
wget -O podkop.ipk https://github.com/itdoginfo/podkop/releases/download/0.7.22/podkop-v0.7.22-r1-all.ipk
wget -O luci-app-podkop.ipk https://github.com/itdoginfo/podkop/releases/download/0.7.22/luci-app-podkop-v0.7.22-r1-all.ipk
wget -O luci-i18n-podkop-ru.ipk https://github.com/itdoginfo/podkop/releases/download/0.7.22/luci-i18n-podkop-ru-0.7.22.ipk

opkg install podkop.ipk
opkg install luci-app-podkop.ipk
opkg install luci-i18n-podkop-ru.ipk

reboot

После установки на роутере останется еще около 26 мб памяти.
