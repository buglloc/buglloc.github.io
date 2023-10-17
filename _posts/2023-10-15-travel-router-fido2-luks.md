---
title: "Travel router: FIDO2 + LUKS in OpenWrt"
classes: wide
categories:
  - home-infra
---
Уж к концу-то 23го года польза от дорожного роутера должна быть очевидна любому истинному шотландцу, но ко мне это осознание пришло буквально в прошлом году. 
Видимо, нужно было кожей почувствовать :) Ну а какое дорожное устройство без шифрования хранящихся на нем данных? О том как я дружил [FIDO2](https://fidoalliance.org/fido2/) [HMAC Secret Extension](https://fidoalliance.org/specs/fido-v2.0-id-20180227/fido-client-to-authenticator-protocol-v2.0-id-20180227.html#sctn-hmac-secret-extension)  с [LUKS](https://gitlab.com/cryptsetup/cryptsetup) контейнером при запуске [OpenWrt](https://openwrt.org/) и будет этот пост.


![](/assets/images/posts/glinet-fido2/cover.jpg){: .align-center}

# Чтоа?!

Дабы проникнуться идеей путешествующего роутера я в прошлом году обзавелся [Slate AX (GL-AXT1800)](https://www.gl-inet.com/products/gl-axt1800/) от GL.iNet и с тех пор не могу отделаться от мысли насколько же кайфово в очередной гостишке достать роутер, подключить его к местному WiFi и получить работающую сеть для всех своих устройств разом. Причем, не просто сеть, а такую как я хочу - с работающим mDNS для Chromecast, с DoH для всех клиентов, с заготовленными туннельчиками на все случаи жизни и т.д.
<figure style="width: 320px" class="align-right">
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/posts/glinet-fido2/axt1800_small.png" alt="">
  <figcaption><center>GL-AXT1800: 125 x 82 x 36mm (245g)</center></figcaption>
</figure>



Если говорить предметно о Slate AX, то в интернетах можно найти разных обзоров (e.g. [GL-iNet Slate AX WiFi 6 gigabit wireless travel router review](https://the-gadgeteer.com/2022/09/24/gl-inet-slate-ax-wifi-6-gigabit-wireless-travel-router-review-safety-and-savings-on-the-road/)). Я ни повторяться ни продавать не хочу, поэтому лучше расскажу почему я в свое время выбрал его:
  - размер и вес не многим больше флагманских лопат
  - достойное железо, особенно для путешественника
  * есть USB Type-A, который можно использовать для всякой периферии
  - питается от USB Type-C, ненавижу возить лишние и не универсальные задярки
  - есть поддержка SD карточки, на которую можно закинуть кинчики и стримить с помощью DLNA на телефон/ноут/chromecast/etc. Конечно, вместо DLNA я бы предпочел Jellyfin, но роутеру GPU не положено :(
  - имеет настолько простой UI, что его не страшно давать даже не подготовленным или уставшим людям
  - основан на OpenWrt, пусть и на версии 21.02, пусть и с бесконечным количеством патчей
  - [до не давних событий был достаточно открытым](https://forum.gl-inet.com/t/formal-statement-on-gl-inet-software-going-closed-source/33288)

Два последних пункта, правда, наталкивают меня на мысль не поискать ли хорошую WiFi карточку (это сложнее, чем кажется) для [NanoPi R5C](https://www.friendlyelec.com/index.php?route=product/product&product_id=290) и собрать сильно более открытый роутер, но речь сегодня не об этом. Во всем остальном он меня полностью устраивает. Во всем остальном, кроме одного - он не имеет шифрования диска с чем я не намерен мириться ◟(`ﮧ´ ◟ )

# Но зачем?!
А зотем! Вы, наверное, не поверите, но я хочу пошифровать данные не в приступе паранойи. Все куда проще - я НЕ хочу в спешке менять явки и пароли, хранящиеся в роутере, потому что мой багаж задержали/утеряли/украли/etc. И это, наверное, максимум, чем тут может помочь шифрование диска. Т.е. логично, что если какой-то злой дяденька имел физический контакт с твоим устройством - его нужно сжечь и больше не использовать. Во-первых, это больше не твоя железка, а во-вторых, ну фу! К счастью, обычно `ЖЕЛЕЗКА < ХРАНИМЫЕ КРЕДЫ`, потому не велика беда, особенно когда ты под пальмой и в отпуске (^_^)

# А как?!
А собсно все уже было придумано до меня:
  - OpenWrt из коробки поддерживает так называемый [extroot](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration)
  - какой-то добрый человек описал костыли для шифрования оного:  [LUKS encrypted extroot](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration#luks_encrypted_extroot)

Насколько я понимаю, основное предназначение extroot в OpenWrt - расширить rootfs за счет какого-то внешнего накопителя в случае, если встроенная флешка не вместительнее наперстка. И в этом ему очень помогает использование [OverlayFS](https://docs.kernel.org/filesystems/overlayfs.html). В стоке он состоит из read-only раздела `rootfs`, поверх которого накладывается writable `rootfs_data`/`ubifs`. Таким образом система живет в `ro` части, а конфиги/доп. пакетики/etc в `rw`. Меняем раздел для `rw` части на кастомный - хоба, вот и extroot :)

В моем случае Slate AX оборудован 128MB NAND Flash, из которой пользователю доступно меньше половины:
```
root@GL-AXT1800:~# cat /proc/mtd
dev:    size   erasesize  name
mtd0: 00180000 00020000 "0:SBL1"
mtd1: 00100000 00020000 "0:MIBIB"
mtd2: 00380000 00020000 "0:QSEE"
mtd3: 00080000 00020000 "0:DEVCFG"
mtd4: 00080000 00020000 "0:RPM"
mtd5: 00080000 00020000 "0:CDT"
mtd6: 00080000 00020000 "0:APPSBLENV"
mtd7: 00180000 00020000 "0:APPSBL"
mtd8: 00080000 00020000 "0:ART"
mtd9: 07280000 00020000 "rootfs"
mtd10: 00080000 00020000 "log"
mtd11: 00080000 00020000 "0:ETHPHYFW"
mtd12: 0041e000 0001f000 "kernel"
mtd13: 034ad000 0001f000 "ubi_rootfs"
mtd14: 03339000 0001f000 "rootfs_data"
root@GL-AXT1800:~# echo $((0x03339000/1024))
52452
```
Это не дофига, но для типичных роутерных нужд хватит и в пару раз меньшего объема. Получается, все чего я должен хотеть - пошифрованный extroot, который будет хранить мои боевые конфиги, ключики, пароли и прочие непотребства. Безопаснее с этим роутером уже не будет. Бонусом появляется возможность загрузиться без использования оного, как-будто так и нужно ;)

И я знаю несколько путей достижения этого:
  1. можно носить с собой пошифрованную флешку с конфигами, а ключ для расшифровки хранить в роутере
  2. можно носить с собой нечто, что расшифрует данные во внутреннем хранилище роутера
  3. третья смешная опция

Оба варика хороши тем, что требуют пролюбить оба устройства (e.g. роутер + флешку), а не что-то одно. Официальная документация [LUKS encrypted extroot](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration#luks_encrypted_extroot) + [Disk Encryption](https://openwrt.org/docs/guide-user/storage/disk.encryption) больше про первый путь (но от этого не менее хороша btw). Мне же больше приглянулся второй:
  - занимаем порт только на этапе загрузки, а это всегда хорошо когда он у тебя один :)
  - при желании, можно оставить включенным роутер, а ключ утащить с собой
  - не возим +1 гаджет _только_ для нужд роутера

Осталось определиться с тем, что расшифрует данные. Я, честно говоря, метался между напилить ченить самому (закончить таки Lupa/начать писать Hang) или взять универсальное/готовое. Любопытство во мне вовремя победило и я решил воспользоваться FIDO2 токеном. Благо, большая часть реализаций должна поддерживать [HMAC Secret Extension](https://fidoalliance.org/specs/fido-v2.0-id-20180227/fido-client-to-authenticator-protocol-v2.0-id-20180227.html#sctn-hmac-secret-extension):
> This extension is used by the platform to retrieve a symmetric secret from the authenticator when it needs to encrypt or decrypt data using that symmetric secret. This symmetric secret is scoped to a credential. The authenticator and the platform each only have the part of the complete secret to prevent offline attacks. This extension can be used to maintain different secrets on different machines.

Который как-раз для этого и предназначен.

# О FIDO2 токенах
В интернете не мало бит информации о том, что такое FIDO U2F и FIDO2. Вот рандомный пост, например: [What Is FIDO2 and How Does It Work?](https://hideez.com/en-int/blogs/news/fido2-explained). Для сегодняшней же задачи важнее понимать, что имплементация должна поддерживать расширение `hmac-secret`, которое появилось в CTAP2 спецификации. Поэтому все U2F токены можно сразу отметать, а у остальных узнать список доступных расширений. Из тех, что оказались у меня под рукой:
![](/assets/images/posts/glinet-fido2/fido_devices.excalidraw.png){: .align-center}

`hmac-secret` ожидаемо поддерживают все FIDO2 реализации. Как Open Source реализации токенов [Pico FIDO](https://github.com/polhenarejos/pico-fido) и [OpenSK](https://github.com/google/OpenSK), так и проприетарные [YubiKey 5 Nano](https://www.yubico.com/product/yubikey-5-nano/) (NB: нужна [firmware v5.2.3+](https://www.yubico.com/blog/whats-new-in-yubikey-firmware-5-2-3/)). По идее, в этом же списке должны быть Nitrokey FIDO2, SoloKey и многие другие, но их нет у меня. Правда, возможно, только пока нет, ибо из тех что у меня в наличии мне пока не нравится ни один. Но  [nRF52840 MDK USB Dongle](https://a.aliexpress.com/_mtdHKns) (на фотке он беленький, с литерой перевернутой M) не понравился меньше остальных.

Вот, кстати, OpenSK из `stable` ветки (внимание на `extension strings`):
```
$ fido2-token -I /dev/hidraw19 
proto: 0x02
major: 0x01
minor: 0x00
build: 0x00
caps: 0x05 (wink, cbor, msg)
version strings: U2F_V2, FIDO_2_0
extension strings: hmac-secret
[...]
```

# И как?
А я вам сейчас расскажу как. Ведь пришла пора настраивать роутер (finally!). Ресетим и обновляем, делаем первичную настройку, логинимся по SSH, а дальше...
### OpenWrt: FIDO2 support
Для начала заводим поддержку FIDO2 токенов, т.к. без этого нет смысла продолжать. Так уж вышло, что libfido2 под OpenWrt собирается [без тулов](https://github.com/openwrt/packages/blob/03a69f84bcf95bf2d54aa8c26814dbfe218e706a/libs/libfido2/Makefile#L48), а для 21.02 он еще и древний как чорт. Ну да не беда, OSS же - [форкаем пакетик](https://github.com/buglloc/openwrt-packages/blob/main/fido2-tools/Makefile) и включаем сборку тулов.

Собираем и ставим наши пакетики + поддержку usb:
```
mkdir -p mkdir -p /tmp/newpkgs && cd /tmp/newpkgs
wget https://github.com/buglloc/openwrt-packages/archive/refs/heads/main.zip
unzip -q main.zip 
opkg update
opkg install openwrt-packages-main/bin/*
opkg install libudev-zero kmod-usb-hid
```

Проверяем работоспособность:
```
root@GL-AXT1800:/tmp/newpkgs# fido2-token -L
/dev/hidraw0: vendor=0x1915, product=0x521f (Nordic Semiconductor ASA OpenSK)
```

То, что доктор прописал!

### OpenWrt: LUKS container
Дальше варим контейнер LUKS, поглядывая в гайд [Disk Encryption](https://openwrt.org/docs/guide-user/storage/disk.encryption) от OpenWrt. В качестве контейнера у меня будет использоваться файлик т.к. 1) это удобнее 2) cryptsetup не поддерживает MTD.

Погнали, ставим нужные зависимости и удаляем не нужные:
```
opkg update
opkg install block-mount kmod-fs-ext4 e2fsprogs kmod-crypto-ecb kmod-crypto-xts kmod-crypto-seqiv kmod-crypto-misc kmod-crypto-user cryptsetup
opkg remove --force-depends lvm2
```

Создаем и разблокируем LUKS-контейнер:
```
. /etc/fidele.conf
mkdir -p $(dirname "$FIDELE_LUKS_PATH")
dd if=/dev/zero of="$FIDELE_LUKS_PATH" bs=1 count=0 seek=24M
fidele-new-passphrase | cryptsetup luksFormat --type luks1 "$FIDELE_LUKS_PATH"
fidele-luks-open
```

Проверяем, что у нас что-то получилось:
```
root@GL-AXT1800:~# ls -la /dev/mapper/$FIDELE_LUKS_LABEL
brw-------    1 root     root      252,   0 Oct 16 21:33 /dev/mapper/extroot
```

Тут стоит пояснить несколько вещей. Во-первых, я использую luks1, т.к. у него заголовок 2MiB, против 16MiB у luks2. Конечно он не так фичаст, но 14MiB есть 14MiB. Во-вторых, не удивляйтесь [fidele](https://github.com/buglloc/openwrt-packages/tree/main/fidele) . Это другой важный пакетик, который мы ставили с моего гитхаба. Он как раз и будет клеем между `block-mount` и `fido2-tools`. У него, кстати, настроечки есть:
```
root@GL-AXT1800:~# cat /etc/fidele.conf
# Path to the luks crypto container
FIDELE_LUKS_PATH="/usr/share/fidele/fidele.extroot"

# Luks label to be used while opening
FIDELE_LUKS_LABEL="extroot"

# Path to the FIDO2 assertation template
FIDELE_ASSERT_PATH="/etc/fidele.f2ap"

# FIDO2 relying party id
FIDELE_RP="fidele@buglloc"

# FIDO2 credentials user name
FIDELE_USER_NAME="gabriel"

# Asks the fido2 authenticator for user presence to be enabled or disabled
FIDELE_UP_REQUIRED="true"

# Asks the fido2 authenticator for user verification to be enabled or disabled
FIDELE_UV_REQUIRED="true"

# PIN to use for fido2 authenticator
FIDELE_PIN=""
```

Не останавливаемся!
### OpenWrt: extroot with LUKS
Вот и самая интересная часть - подготовка и монтирование extroot. Тут, стоит обратиться к гайдам [extroot](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration) и [LUKS encrypted extroot](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration#luks_encrypted_extroot) от OpenWrt. Но в двух словах, наш план таков:
  - вызываем [fidele-luks-open](https://github.com/buglloc/openwrt-packages/blob/main/fidele/files/usr/sbin/fidele-luks-open) из [preinit](https://openwrt.org/docs/techref/preinit_mount#preinit) стадии загрузки
  - он открывает наш LUKS-контейнер
  - дальше OpenWrt находит extroot в fstab конфиге _текущего_  `rootfs_data` 
  - делает всю остальную магию по монтированию и сборке overlay с использованием  `rootfs_data` из крипто-контейнера

Не станем откладывать, форматируем LUKS раздел:
```
. /etc/fidele.conf
DEVICE="/dev/mapper/${FIDELE_LUKS_LABEL}"
mkfs.ext4 -L "$FIDELE_LUKS_LABEL" "${DEVICE}"
```

Проверяем (внимание на `/dev/dm-0`):
```
root@GL-AXT1800:~# block info
/dev/mtdblock13: UUID="5fb30cde-6d43fda4-54c9861f-5033727f" VERSION="4.0" MOUNT="/rom" TYPE="squashfs"
/dev/mtdblock9: UUID="434930875" VERSION="1" TYPE="ubi"
/dev/ubi0_2: UUID="634efd63-6628-4bcc-825a-9ad16fa34cfc" VERSION="w4r0" MOUNT="/overlay" TYPE="ubifs"
/dev/loop0: UUID="b7292b54-d130-4ec8-ad26-cf2e1e4ece41" TYPE="crypto_LUKS"
/dev/mmcblk0p1: UUID="b130ec8a-d719-4a93-8bed-73447d87ef83" VERSION="1.0" MOUNT="/tmp/mountd/disk1_part1" TYPE="ext4"
/dev/dm-0: UUID="1174715e-6d5f-4bde-be08-5b977ad31e7e" LABEL="extroot" VERSION="1.0" TYPE="ext4"
```

Добавляем в fstab монтирование нашего extroot:
```
DEVICE="/dev/dm-0"
eval $(block info ${DEVICE} | grep -o -e 'UUID="\S*"')
eval $(block info | grep -o -e 'MOUNT="\S*/overlay"')
uci -q delete fstab.extroot
uci set fstab.extroot="mount"
uci set fstab.extroot.uuid="${UUID}"
uci set fstab.extroot.target="${MOUNT}"
uci set fstab.extroot.options="rw,noatime"
uci commit fstab
```

Проверяем:
```
root@GL-AXT1800:~# uci show fstab.extroot
fstab.extroot=mount
fstab.extroot.target='/overlay'
fstab.extroot.options='rw,noatime'
fstab.extroot.uuid='1174715e-6d5f-4bde-be08-5b977ad31e7e'
```

Копируем текущий `rootfs_data`:
```
. /etc/fidele.conf
mount -t ext4 ${DEVICE} /mnt
echo $(basename "$FIDELE_LUKS_PATH") > /tmp/exclude
tar -X /tmp/exclude -C ${MOUNT} -cvf - . | tar -C /mnt -xf -
```

Добавляем загрузку нужных ядерных модулей в preinit:
```
for mod in "usb-hid" "hid-generic" "30-dm" "09-crypto-hmac" "09-crypto-sha256" "09-crypto-xts" "09-crypto-user"
do
   touch "/etc/modules.d/$mod"
   ln -s "../modules.d/$mod" "/etc/modules-boot.d/$mod"
done
```

Проверяем:
```
root@GL-AXT1800:~# ls -la /overlay/upper/etc/modules-boot.d/
drwxr-xr-x    2 root     root           656 Oct 16 22:00 .
drwxr-xr-x   19 root     root          2136 Oct 16 21:33 ..
lrwxrwxrwx    1 root     root            27 Oct 16 22:00 09-crypto-hmac -> ../modules.d/09-crypto-hmac
lrwxrwxrwx    1 root     root            29 Oct 16 22:00 09-crypto-sha256 -> ../modules.d/09-crypto-sha256
lrwxrwxrwx    1 root     root            27 Oct 16 22:00 09-crypto-user -> ../modules.d/09-crypto-user
lrwxrwxrwx    1 root     root            26 Oct 16 22:00 09-crypto-xts -> ../modules.d/09-crypto-xts
lrwxrwxrwx    1 root     root            18 Oct 16 22:00 30-dm -> ../modules.d/30-dm
lrwxrwxrwx    1 root     root            24 Oct 16 22:00 hid-generic -> ../modules.d/hid-generic
lrwxrwxrwx    1 root     root            20 Oct 16 22:00 usb-hid -> ../modules.d/usb-hid
```

Применяем костыль с подменой `block`:
```
mv /sbin/block /sbin/block.bin
cat << "EOF" > /sbin/block
#!/bin/sh

# Set to "" to disable debug logs
export DEBUG=1

SDIR=${0%/*}
BLOCK="${SDIR}/block.bin"
LD_LIBRARY_PATH=${LD_LIBRARY_PATH:-.}
LD_LIBRARY_PATH="${SDIR}/../usr/lib:${LD_LIBRARY_PATH}"
PATH=$PATH:${SDIR}:${SDIR}/../usr/sbin:${SDIR}/../bin:${SDIR}/../usr/bin

block() {
  exec -a ${0} ${BLOCK} "$@"
}

if [ "$PREINIT" != "1" ]; then
  block "$@"
fi

get_jiffies() {
  head -n3 /proc/timer_list | tail -n1 | cut -d' ' -f 3
}

if [ -z "$BLOCK_LOG" ] && [ -n "$DEBUG" ]; then
  TIME=$(get_jiffies)
  export BLOCK_LOG="/tmp/block.$(printf '%016d' ${TIME:-9999999999}).log"
  exec 2>"$BLOCK_LOG"
  set -x
fi

if [ ! -x "$BLOCK" ]; then
  echo "Error: ${BLOCK} is not an executable" >&2
  return 1
fi

if [ "$1" = "extroot" ] && [ -e ${SDIR}/../.use_crypt_extroot ]; then
  # We are being called to setup the extroot, so make sure crypto block
  # devices are all setup.

  # Hotplug runs too late, create device nodes for /dev/hidraw*, if there are any
  for SYSDEVPATH in /sys/class/hidraw/*; do
    [ ! -f "$SYSDEVPATH"/dev ] && continue
    [ -e "/dev/${SYSDEVPATH##*/}" ] && continue
    MAJMIN=$(cat "$SYSDEVPATH"/dev | tr ':' ' ')
    mknod -m0600 /dev/${SYSDEVPATH##*/} c $MAJMIN
  done

  ALTROOT="${SDIR}/.." "$SDIR"/../usr/sbin/fidele-luks-open
fi

block "$@"
EOF

chmod +x /sbin/block
```

И последним шагом создаем файлик `/.use_crypt_extroot` в _текущем_  `rootfs_data`  - это наш стоп-кран:
```
touch /.use_crypt_extroot
```

На всякий случай, перед перезагрузкой стоит проверить работоспособность конструкции:
```
export PREINIT=1
mount_root
```

FIDO2 токен должен попросить его потрогать и `/dev/dm-0` смонтироваться в `/overlay`:
```
root@GL-AXT1800:/rom/root# cat /etc/mtab 
mtd:ubi_rootfs /rom/rom squashfs ro,relatime 0 0
proc /proc proc rw,nosuid,nodev,noexec,noatime 0 0
sysfs /sys sysfs rw,nosuid,nodev,noexec,noatime 0 0
tmpfs /tmp tmpfs rw,nosuid,nodev,noatime 0 0
/dev/ubi0_2 /rom/overlay ubifs rw,noatime 0 0
overlayfs:/overlay /rom overlay rw,noatime,lowerdir=/,upperdir=/overlay/upper,workdir=/overlay/work 0 0
tmpfs /dev tmpfs rw,nosuid,relatime,size=512k,mode=755 0 0
devpts /dev/pts devpts rw,nosuid,noexec,relatime,mode=600,ptmxmode=000 0 0
debugfs /sys/kernel/debug debugfs rw,noatime 0 0
bpffs /sys/fs/bpf bpf rw,nosuid,nodev,noexec,noatime,mode=700 0 0
pstore /sys/fs/pstore pstore rw,noatime 0 0
/dev/mmcblk0p1 /tmp/mountd/disk1_part1 ext4 rw,relatime,data=ordered 0 0
nfsd /proc/fs/nfsd nfsd rw,relatime 0 0
/dev/dm-0 /rom/mnt ext4 rw,relatime,data=ordered 0 0
/dev/dm-0 /overlay ext4 rw,noatime,data=ordered 0 0
overlayfs:/overlay / overlay rw,noatime,lowerdir=/,upperdir=/overlay/upper,workdir=/overlay/work 0 0
```

Фуф, выглядит как успех! Теперь-то можно ребутаться, проверять что все успешно смонтировалось и донастраивать роутер уже в пошифрованном руте.

# In action
Самое время проверить делом, я этого долго ждал :) Для наглядности я завел две AP - `GoodPotato` и `BadPotato`, одна в пошифрованном extroot, вторая нет. 

Вот загрузка с разблокировкой LUKS-контейнера:
![](/assets/images/posts/glinet-fido2/good-boot.png){: .align-center}

И без:
![](/assets/images/posts/glinet-fido2/bad-boot.png){: .align-center}

КРАСОТА ясчитаю!

## Final thoughts
Как и всегда с такими постами, может показаться, что как-то все слишком уж легко, но это не так. Если честно, то совсем не так, поэтому не отчаивайтесь если при попытке повторить что-то будет не взлетать - это норма :)
Если говорить про получившийся результат, то я скорее остался доволен. Да, в [fidele](https://github.com/buglloc/openwrt-packages/tree/main/fidele) очень есть что улучшить (а где нет? хех). Да, имеющиеся FIDO2 токены далеки от моего идеала. Но это все явно не тянет на deal-breaker. Главное, что сам концепт жизнеспособен и просто требует еще чуточку внимания и любви (как и все мы ^^).

Всем благ (づ˶•༝•˶)づ♡