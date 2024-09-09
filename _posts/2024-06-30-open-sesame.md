---
title: "Open Sesame: PinkyWinky"
classes: wide
header:
  og_image: /assets/images/posts/open-sesame/PinkyWinky.png
categories:
  - iot
  - esphome
redirect_from:
  - /iot/esphome/open-sesame/
---

Внезапным для самого себя образом обнаружил, что лучший подарок молодому IoT-строителю - переезд в дом %) Серьезно, с самого порога получаешь какое-то невероятное количество ~~проблемок~~ задачек, которые так и зудит порешать. В этот раз сказ о входных воротах.
Внезапным для самого себя образом обнаружил, что лучший подарок молодому IoT-строителю - переезд в дом %) Серьезно, с самого порога получаешь какое-то невероятное количество ~~проблемок~~ задачек, которые так и зудит порешать. В этот раз сказ о входных воротах.

![gate](/assets/images/posts/open-sesame/PinkyWinky.png)

Внутри использование [ESPHome](https://esphome.io/) и [CC1101](https://www.aliexpress.com/item/1005002074380868.html), кастомная прошивка для бикона на [nRF52810](https://www.nordicsemi.com/Products/nRF52810) и многое другое. Пристегивайтесь :)

## Но сперва..._на кой чёрт_?
Вопрос, который должен бы задать себе любой человек перед тем, как приступить решать какую-либо проблемку. В моем случае проблема звучит так:  жена, как и огромное количество людей в Тае, водит скутер и ей нужно открывать/закрывать входные ворота. Да, вот так просто. Вот они, кстати:
![gate](/assets/images/posts/open-sesame/gate.jpg)

К счастью, они умеют открываться с брелока. Но, традиционно, брелок этот какая-то поделка, которой и стремно, и не удобно пользоваться будучи _за рулем скутера_.

Хмм, починить то, что не было сломано - бывает ли задачка достойнее? Я тоже так подумал и начал вынашивать план, как сделаю жену чуть счастливее :)

## Планчик
Управление воротами у меня бесхитростное, работает на частоте 330Mhz, никаких [rolling codes](https://en.wikipedia.org/wiki/Rolling_code) и прочих безопасностей не имеет. Посему, вариантов приделать более удобное управление огромное множество. Я решил так:
  - берем BLE Beacon, лепим его на скутер аки кнопку
  - дома включаем BLE сканер у ESP32 для отслеживания бикона
  - к этому же ESP32 цепляем RF передачик
  - отправляем сигнал блоку управления воротами при нажатии на кнопку
  - профит

С одной стороны у нас получается довольно локальная история без зависимостей от [Home Assistant](https://www.home-assistant.io/), интернетов и т.д. С другой - поуправлять воротами из того же Home Assistant все-таки будет возможно. А с третьей - бикон должен позволить в будущем автоматизировать открытие ворот (но об этом в другой раз, не все штуки приехали). Ну и во все том же будущем подключить бикончик к эпловой (см. [OpenHaystack](https://github.com/seemoo-lab/openhaystack/)) или лучше гугловой "Find My Device" сети, всяк лишним не будет. В общем, у меня на него большие планы, а пока побудет кнопкой ;)

## RF transmitter
Наверное, никого не удивлю, если скажу, что люди [решали](https://community.home-assistant.io/t/esphome-gate-remote/550111) эту [задачу](https://www.instructables.com/Decoding-and-sending-433MHz-RF-codes-with-Arduino-/) [сотню другую](https://github.com/nopnop2002/esp-idf-rc-switch) [раз](https://github.com/julienfouilhe/automate-gate-opening). Поэтому тут довольно сложно быть оригинальным - берем ESP32 + RF передатчик + ESPHome и лепим то, что пожелаем :)

А значит и затягивать нет смысла, выбор железа у меня таков:
  - [Seeed Studio XIAO ESP32-S3](https://wiki.seeedstudio.com/xiao_esp32s3_getting_started/) как основа всего
  - [CC1101 Wireless Module](https://www.aliexpress.com/item/1005005105272262.html) в роли RF передатчика
  - [2.4G Antenna](https://www.aliexpress.com/item/1005006179442651.html)
  - Lithium Battery 3.7v дабы не ресетить стейт от пропажи питания
  - какая-то коробушка

ESP32-S3 я взял в-первую очередь ради поддержки Bluetooth 5 (LE), а во-вторую за его два ядра, опасаясь возможных проблем с сожительством BLE и WiFi (см. [ESP32 BLE Tracker: Use on single-core chips](https://esphome.io/components/esp32_ble_tracker#use-on-single-core-chips)).

А вот СС1101 взял наугад. В мире существует множество разных RF передатчиков с поддержкой 330Mhz (e.g. SYN115, SI4432 и т.д.), СС1101 мне показался достаточно популярным, чтобы не промахнуться - вродь не ошибся :D

## RF transmitter: дела хардварные
С жезелом определились, время подключать:
```
| ESP32-S3    |   СС1101 module   |
|-----------------|---------------------------|
|      GND       |             GND             |
|  VCC  (3v3)  |             VCC              |
|    GPIO9      |              TX                |
|    GPIO8      |             SCK              |
|    GPIO7      |            MISO            |
|    GPIO1      |              CS               |
|    GPIO2      |            MOSI            |
|    GPIO4      |              RX               |
```

Для визуалов:
![gate-rf-schema](/assets/images/posts/open-sesame/gate-rf.png)
Запаиваем и корпусируем:
![gate-rf](/assets/images/posts/open-sesame/gate-rf-welding.png)

## RF transmitter: дела софтварные
Фуф, запаяли, можно наконец-то заняться софтом. Как я и говорил, CC1101 довольно популярный чип, поэтому найти что-то готовое для его применения в ESPHome не составило труда, например:
  - [esphome PR: cc1101 radio transmitter component](https://github.com/esphome/esphome/pull/6300)
  - [github://dbuezas/esphome-cc1101](https://github.com/dbuezas/esphome-cc1101)
  - слышал об успешном использовании [github.com://jgromes/RadioLib](https://github.com/jgromes/RadioLib)

Я выбрал PR для ESPHome в надежде, что его допинают до релиза и он станет доступен из коробки. Кмк, полезнее был бы порт RadioLib, но и CC1101 в составе ESPHome тоже неплохо :)

Дело за малым:
  - немного программируем на ямлах: [конфиг для ESPHome](https://gist.github.com/buglloc/1bfffa121f99a481a878c4a52f70bce2)
  - прошиваем им наш RF transmitter
  - добавляем его в Home Assistant
  - проверяем отправляемость сигнала:
    * я, как хлебушек, пользуюсь Flipper Zero для этого
    * запускаем запись Sub-GHz сигнала (`Sub-GHz` -> `Read RAW`)
    * жмем на кнопку `Open Gate` в Home Assistant:
![test-cc1101](/assets/images/posts/open-sesame/gate-rf-test.png)
  - радуемся, ведь все выглядит рабочим, и мы можем двигаться дальше

## RF transmitter: дела сигнальные
А дальше нас ждет увлекательное путешествие в мир радио, ведь нам нужно отправить правильный сигнал блоку управления воротами.

Скорее всего крепкие специалисты пользуются чем-то более подходящим (а то и самим CC1101, [например](https://github.com/dbuezas/esphome-remote_receiver-oscilloscope)), я же опять расчехляю Flipper Zero:
  - запускаем запись Sub-GHz сигнала (см. [Sub-GHz: Reading RAW signals](https://docs.flipper.net/sub-ghz/read-raw))
  - жмем кнопку брелока и прикапываем ее сигнал:
![read-button](/assets/images/posts/open-sesame/key-signal.jpg)
  - закидываем в [плоттер](https://lab.flipper.net/pulse-plotter)
  - видим явные периоды отправки кода:
![flipper-pulse](/assets/images/posts/open-sesame/flipper-pulse-0.png)
  - и сам код при более детальном рассмотрении:
![flipper-pulse](/assets/images/posts/open-sesame/flipper-pulse-1.png)
  - осталось записать конечный код посчитав нолики и единички
  - прикинуть продолжительность пульса, описать как выглядит sync (`1 + 0*21`),  `0` (`1000`) и  `1` (`1110`)

Вуаля, через пару итераций "записываем отправляемый сигнал, кидаем в плоттер, сравниваем с оригиналом" получаем итоговый конфиг открытия ворот:
```yaml
button:
  - platform: template
    name: "Open gate"
    id: gate
    on_press:
      - remote_transmitter.transmit_rc_switch_raw_cc1101:
          code: '0100011111111100101110101'
          protocol:
            pulse_length: 345
            sync: [1, 21]
            zero: [1, 3]
            one: [3, 1]
          repeat:
            times: 4
            wait_time: 0s
```


Вот и все, пока можно отложить ESPHome. Дальше нам очень нужно разобраться с биконом.

## PinkyWinky
С биконом, пожалуй, самая интересная история всего проекта. На всякий случай напомню, что зачастую бикон - это манюсенькое устройство, которое что-то броадкастит по Bluetooth. Причем, броадкастить он может примерно что угодно:
  - это может быть как просто UUID для отслеживания положения, как в [iBeacon](https://en.wikipedia.org/wiki/IBeacon) 
  - так и ссылка/EID, как в случае с [Eddystone](https://en.wikipedia.org/wiki/Eddystone_(Google))
  - так и любая нужная вендору информация. Например, состояние кнопки, уровень заряда батарейки и т.д.

В общем, отличная штуковина для нашей задачки. Но есть нюанс - все биконы, которые я держал в руках, были очень ватной кнопкой в connectionless режиме :( А я хотел именно connectionless режим, чтобы позволить себе терять пакетики при подъезде к дому (сложно, знаете ли, точно на глазок определить пора ли жать на кнопку)...

Вечер перестает быть, томным подумал я! Засучил рукава, открыл AliExpress и принялся искать бикон, для которого было бы удобно писать кастомную прошивку. Вынырнул я из поиска с заказом двух BLE биконов от Holyiot на [nRF52810](https://www.nordicsemi.com/Products/nRF52810):
![beacon](/assets/images/posts/open-sesame/Holyiot-nRF52810-beacon.png)

C [акселометром](https://www.aliexpress.com/item/1005007002557705.html) и [без](https://www.aliexpress.com/item/1005004501422185.html), так уж вышло. На Summer Sale оба красавчика обошлись мне примерно в $5 каждый. Выбрал я их по нескольким причинам:
  - SOC [nRF52810](https://www.nordicsemi.com/Products/nRF52810) :
    * выполнен на базе Cortex-M4 64 MHz
    * поддерживает Bluetooth Low Energy
    * имеет на борту 192 kB флешки и 24 kB RAM
    * поддержан в [Zephyr Project](https://www.zephyrproject.org/), на котором и будем писать прошивку
  - обещает быть waterproof
  - на платке есть выводы GND/SWCLK/SWDIO для прошивки и отладки

В общем, ТТХ идеально подходят для нашей будущей кнопки. А пока биконы ехали ко мне из Китая, я покопался в столе и вытащил [XIAO nRF52840](https://wiki.seeedstudio.com/XIAO_BLE/) + [Expansion Board Base](https://wiki.seeedstudio.com/Seeeduino-XIAO-Expansion-Board/), которые купил на какой-то из распродаж до этого:
![xiao-nrf52840](/assets/images/posts/open-sesame/xiao-nrf52840.jpg)
И принялся писать прошивку. Почему на [nRF52840](https://www.nordicsemi.com/Products/nRF52840)? Это самое близкое к nRF52810, что у меня было + отлаживаться на nRF52840 сильно более приятно, т.к. он имеет целых 256 kB RAM против 24 kB у nRF52810 (которых, как оказалось, для BLE стека в Zephyr прям впритирку).

## PinkyWinky: firmware
Сам [PinkyWinky](https://github.com/buglloc/pinky-winky) написан на C поверх [Zephyr Project](https://www.zephyrproject.org/). Почему Zephyr? Давно хотел попробовать, а тут такой шанс. Если в двух словах, то Zephyr - real-time OS для [кучи железок](https://docs.zephyrproject.org/latest/boards/index.html), имеет поддержку devicetree, bluetooth, многозадачности и прочих ништяков. Приятный проект, мне понравился :)

Логика работы __PinkyWinky__ предельно проста, как то и должно быть:
  - на старте генерирует случайное начальное значение таймстемпа (`initial_ts`)
  - запускает Legacy BLE Advertising (Extended Advertising не поддерживает ESPHome)
  - в manufacturer specific data передает:
    * версию (`version`) - 1 байт
    * заряд батарейки (`batt_percent`) - 1 байт
    * состояние кнопки (`btn_pressed`) - 1 байт
    * текущий таймстемп (`ts`) - 4 байта
    * укороченая sha1 подпись (`sign`) - 10 байт
  - при нажатии на кнопку:
    * устанавливает `btn_pressed = true`
    * устанавливает `ts = initial_ts + seconds_since_boot`
    * зажигает LED
    * перезапускает BLE Advertising для большей реактивности
    * через какое-то время (1s по умолчанию):
      - сбрасывает состояние `btn_pressed` и LED
      - устанавливает `ts = initial_ts + seconds_since_boot`
  - периодически (15s по умолчанию) устанавливает `ts = initial_ts + seconds_since_boot`

Как то часто бывает, многое тут не случайно. Так таймстемп и его ротация служат защитой от копирования и заметного усложнения эксплуатации [Replay](https://ivanorsolic.github.io/post/car-hacking/), [Rolljam](https://www.andrewmohawk.com/2016/02/05/bypassing-rolling-code-systems/) и аналогичных атак. Рандом в `initial_ts` нужен, т.к. у бикона нет источника времени. А подпись укорочена до 10 байт, потому что в Legacy BLE Advertising доступно лишь 31 (adv data) + 31 (scan response) байт данных на все про все, вместе с именем, флажками и т.д. При этом риски намутить коллизию на 7 байтах оцениваю как минимальные. Вместо подписи можно было бы зашифровать данные, но мне показалось лишним (скрывать там нечего, а бояться отслеживания с партией устройств в 1 штуку глупо).

Пора. Пора прошивать XIAO nRF52840, запускать [nRF Connect](https://play.google.com/store/apps/details?id=no.nordicsemi.android.mcp&hl=en&pli=1) и проверять:
![xiao-nrf52840-test](/assets/images/posts/open-sesame/xiao-nrf52840-test.png)

## PinkyWinky: ESPHome
Конечно, вся эта кастомщина в BLE Advertising не может обойтись без соответствующей поддержки в ESPHome. Так, наряду с прошивкой для бикона пришлось написать и компонент к ESPHome - [pinky_winky](https://github.com/buglloc/esphome-components/tree/main/components/pinky_winky), который:
  - реализует логику парсинга, валидации и проверки подписи данных от PinkyWinky
  - умеет прикапывать `initial_ts` в [Non-Volatile Storage](https://docs.espressif.com/projects/esp-idf/en/v5.2.2/esp32s3/api-reference/storage/nvs_flash.html), дабы переживать рестарты
  - позволяет сбросить  `initial_ts` при смене батарейки у PinkyWinky
  - предоставляет [Binary Sensor](https://esphome.io/components/binary_sensor/index.html) для кнопки
  - предоставляет [Sensor](https://esphome.io/components/sensor/index.html) для батарейки

Но всегда лучше один раз увидеть, потому пробуем проверить input lag:
  - еще немного попрограммируем на ямлах: [конфиг для ESPHome](https://gist.github.com/buglloc/90e618bf0dc6768aa1746aba8f1b1ecb)
  - тестим:
{% include video id="8mbDp18yUbw" provider="youtube" %}
  - нраица! Самое время катить в прод.

## PinkyWinky: Holyiot
Уф, полевые испытания все ближе! А пока портируем прошивку под наш бикон. Благо, благодаря Zephyr Project, все что для этого нужно - описать новый борд (см. [Board Porting Guide](https://docs.zephyrproject.org/latest/hardware/porting/board_porting.html)): [Holyiot 21014](https://github.com/buglloc/pinky-winky/tree/main/boards/holyiot/holyiot_21014). Ничего хитрого.

Кстати, помните я говорил, что памяти у nRF52810 впритирку для BLE стека Zephyr? Вот какой-никакой пруф со сборкой PinkyWinky (тут отключено логирование, serial console, не используемые фичи bluetooth и пр.):
```
~/zephyr/apps/pinky-winky on main
% west build -p always -b holyiot_21014
...
...
Memory region         Used Size  Region Size  %age Used
           FLASH:      104440 B       192 KB     53.12%
             RAM:       21208 B        24 KB     86.30%
        IDT_LIST:          0 GB        32 KB      0.00%
```

Ну все, осталось совсем немного. Припаяться к платке:
![holyiot-welding](/assets/images/posts/open-sesame/holyiot-welding.png)

Прошить нашу прошивочку:
![holyiot-flashing](/assets/images/posts/open-sesame/holyiot-flashing.png)

Если интересно, то для прошивки и отладки я перешел на [Black Magic Probe](https://black-magic.org/) поверх [WeAct Studio Black Pill F4](https://github.com/WeActStudio/WeActStudio.MiniSTM32F4x1). Хз как жил без него, если честно. Получать доступ в GDB server с автоопределением таргета без плясок с запуском скриптов для OpenOCD и прочего  - это какая-то чистая магия :)
 
И подсобрать все обратно с небольшим модом для более короткого хода кнопки.
![holyiot-asembly](/assets/images/posts/open-sesame/holyiot-asembly.png)

## Final test
Yay! Пора пробовать в бою:
  - в последний [на сегодня] раз программируем на ямлах: [конфиг для ESPHome](https://gist.github.com/buglloc/fc95593d7cb858fdaada9fea3265cf70)
  - и тестим:
{% include video id="1b6GQJ5LkOk" provider="youtube" %}
  - тааа-дааам, работает как и планировалось :)

Дальше по планам проверсти больше полевых испытаний, чёнить помелочи поправить и уже зимо-осенью заняться автоматизацией открытия ворот и подключению к Find My Device. Да не скоро, но вы все равно не переключайтесь ;)

А пока у меня все, всем кота (づ˶•༝•˶)づ♡
