---
title: "WasherNotifier: the journey"
classes: wide
toc: false
categories:
  - iot
---

Так уж вышло, что в один солнечный день я осознал, что больше жить не могу без нотификаций от стиральной машины. Все дело в том, что в текущей квартире она стоит на балконе, и о ней как-то очень уж просто забыть или не услышать вовсе. Как итог, забытые вещи и горе в семье.
![](/assets/images/posts/wn/cover.excalidraw.png){: .align-center}

### User story
Слышал, что без user story обречен почти любой проект, поэтому я не стал рисковать и представил себе идеальный flow:

![](/assets/images/posts/wn/the-plan.excalidraw.png){: .align-center}

Ага, ок. Звучит так, что нам нужно:
  - узнать, когда стиральная машина закончила работку
  - отправить нотификацию в приложеньку/часики/etc

В идеале еще узнавать бы, забрал ли кто вещи, т.к. пользователей у неё целых два. Но об этом позже. Сейчас бы узнать когда работа машины завершена. Для себя я это назвал как Washer Running State, или WRS.

### WRS
Немного гуглежа, и список вариантов решений (скорее всего неполный) был у меня на столе. И так, можно было сделать:
  1. по-богатому: купить стиральную машину с wifi, meh :)
  2. по-рукастому: модифицировать стиральную машину
  3. по-модному: прикрутить sound recognition или computer vision
  4. по-классике: замерить потребление энергии
  5. по-всратому: прикрутить датчик вибрации

И мысли по этому поводу в тот момент времени:
- по-богатому как-то слишком уж бессмысленно и беспощадно для съемной квартиры
- рукастый варик отъехал по этой же причине - техника не моя + я довольно рукожопый
- sound recognition хоть и звучит неплохо (кек ^^), но профит выглядит мелковатым для потраченных усилий
- для computer vision не придумал, куда бы на балконе прикрутить камеру (например, ченить в духе [ESP32 Cam](https://www.aliexpress.com/item/1005005447654263.html)). Видимо в другой раз ;)

Остались два варика, которые я и решил реализовать. Приступим!

#### WRS: vibration sensor
Начать я, конечно же, решил с всратого. Ничего не могу с собой поделать, всратости попадают мне в самое сердечко :)

В трех словах весь замысел заключается в:
  - подключаем датчик вибрации к MCU
  - когда датчик вибрации сработает, он отправит сигнал
  - если движение было, а потом пропало - стиральная машина (скорее, сушилка в ней) закончила свою работу

Для этого достал из шкафа [ESP32-DevKitC](https://a.aliexpress.com/_m01z9n0), который часто использую для тестов, и заказал с алика vibration sensor [SW-18010P](https://a.aliexpress.com/_mOudBHG) за $1.6 (можно было бы и SW-420, который раза в три дешевле, но я решил как решил).

Подключил сию нехитрую конструкцию:
```markdown
| ESP32 | SW-18010P |
|-------|-----------|
|  3.3V |     +     |
|  GND  |     -     |
|  D34  |    Dout   |
```
![](/assets/images/posts/wn/esp32-sw18010p.png)

Написал&&Прошил минимальный конфиг для [ESPHome](https://esphome.io/) ([сорец](https://gist.github.com/buglloc/3ea4cf04a89e318fb58cc6d86886bbe0#file-vibration-esp-yaml)):
```yaml
esphome:
  name: dev01
  friendly_name: Dev01

esp32:
  board: esp32dev
  framework:
    type: arduino

api:
  encryption:
    key: "DEADBEEF"

ota:
  password: "deadbeef"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  manual_ip:
    static_ip: 10.11.9.1
    gateway: 10.11.0.1
    subnet: 255.255.0.0
    dns1: 10.11.1.2

# Больше логов богу логов
logger:

# Не люблю много думать о времени, потому всегда использую Home Assistant Time Source
# doc: https://esphome.io/components/time/homeassistant.html
time:
  - platform: homeassistant
    id: homeassistant_time

sensor:
  # WiFi Signal Sensor, дабы лучше понимать что с WiFi
  # doc: https://esphome.io/components/sensor/wifi_signal.html
  - platform: wifi_signal
    name: "WiFi Signal Strength"
    update_interval: 60s

   # Pulse Counter Sensor с медианой, т.к. у меня стиральная машина стоит на балконе
   # А там бывает и кот и птицы и люди, хотелось не реагировать на случайные срабатывания
   # doc: https://esphome.io/components/sensor/pulse_counter.html
   # doc: https://esphome.io/components/sensor/#median
  - platform: pulse_counter
    name: "Washer vibration pulse rate"
    internal: false
    id: vibration_pulse_rate
    icon: mdi:vibrate
    update_interval: 250ms
    pin:
      number: GPIO34
      mode: input
    filters:
    - median:
        window_size: 120 # скользяще окно около 30 секунд
        send_every: 4 # отправляем раз в секунду
        send_first_at: 3

binary_sensor:
  # Status Binary Sensor, дабы отрисовывать аптайм
  # doc: https://esphome.io/components/binary_sensor/status.html
  - platform: status
    name: Status

  # Template Binary Sensor, который будет зажигать лампочку на основе ответа лямбды
  # Так же устанавливаем ему "delayed_on_off" для минимизации фолзов
  # doc: https://esphome.io/components/binary_sensor/template.html
  # doc: https://esphome.io/components/binary_sensor/index.html#delayed-on-off
  - platform: template
    name: "Washer Running"
    device_class: vibration
    filters:
      - delayed_on_off:
          time_on: 20ms
          time_off: 5min # сглаживаем отпутствие вибрации
    lambda: |-
      return id(vibration_pulse_rate).state >= 240;

# Restart Switch, чтобы мочь рестартить MCU из HA
# doc: https://esphome.io/components/switch/restart.html
switch:
  - platform: restart
    name: Restart
```

Добавил его в Home Assistant:
![](/assets/images/posts/wn/initial-esp-vibr-in-ha.png)

На весу проверил работоспособность и осуществил ~~сопле-~~ монтаж в тайском стиле:
![](/assets/images/posts/wn/esp-vibr-montage.jpg)

Приемлемо? Приемлемо!
#### WRS: power consumption
Но, не смотря на то, что концепция с датчиком вибрации мне довольно сильно приглянулась, я ожидал наличие проблем. Из очевидных:
 - должно быть сложно избавиться от всех false negative/positive, особенно с тусичами котейки на балконе
 - вибрация зависит от слишком большого числа параметров - от программы стирки до, что хуже, загруженности машины

Поэтому вместе с датчиком вибрации я решил прикупить и какую-нить умную розетку с мониторингом потребления энергии, чисто чтоб запустить их вместе и не наблюдать за стиркой x2 времени. Но тут меня ждал сюрприз:
![](/assets/images/posts/wn/socket-surprise.jpg)
Да-да, это [Type-O](https://www.worldstandards.eu/electricity/plugs-and-sockets/o/), который никому за пределами Таиланда не интересен. Если вы увидите шутника про появление 15го стандарта у программистов - выдайте ему затрещину и напомните про электриков. Про электриков, которые не смогли договориться о количестве штырьков, их размере и расположении.
В Таиланде зачастую можно встретить [type `O` / `C` / `A`/ `B`](https://www.worldstandards.eu/electricity/plug-voltage-by-country/thailand/) (иногда еще [I](https://www.worldstandards.eu/electricity/plugs-and-sockets/i/) кстати), и type-O худшая из карт для поиска умной розетки :( Но деваться некуда, отыскал приемлемую с виду розетку [SMATRUL 20A/16A Tuya/Smart life WiFi Socket](https://www.lazada.co.th/products/smatrul-20a16a-tuyasmart-life-wifi-socket-universal-us-eu-smart-plug-adapter-power-monitor-european-plug-wireless-remote-voice-timer-for-google-homealexatmall-genie-i2911432612-s10659100651.html?dsource=share&laz_share_info=674305705_9_100_100322224233_674305706_null&laz_token=88cc5f36711df9024f45cb3f493e2031&exlaz=e_iqeJxmuQOjPGip8qo24MCSTeQCjDld4siVQ4kY1ODqrCXvTKdNAtBb55%2BgIiSkmNnXkPoNuoMIIpR8IEI%2FLi1U0sC6tUOx2TD3WnASIas1Y%3D&sub_aff_id=social_share&sub_id2=674305705&sub_id3=100322224233&sub_id6=CPI_EXLAZ), которая обошлась мне примерно в $4:
![](/assets/images/posts/wn/smatrol.jpg){: .align-center}


Из картинки, собсно, все спеки и ясны :) Я сильно рассчитывал, что прошивка окажется какой-то древней, и я прошью туда  [Tasmota](https://tasmota.github.io/docs/) или [ESPHome](https://esphome.io/) по воздуху с помощью [tuya-convert](https://github.com/ct-Open-Source/tuya-convert). Но обломался :( В историю с _аккуратненько_ разобрать розетку (собранную прессом, на минуточку!) я совсем не верю, поэтому осталось только воспользоваться [localtuya](https://github.com/rospogrigio/localtuya), который без проблем завелся:
![](/assets/images/posts/wn/localtuya.png)
Выглядит рабочим. Время втыкать вилку в розетку, розетку в розетку и проверять в деле.

#### WRS: IRL
Хе, коль уж оба варианта реализованы и задеплоены, самое время начинать полевые испытания. Пожалуй, это были самые интересные постирушки в моей жизни :)

Что ж, запускаем и оцениваем:
![](/assets/images/posts/wn/first-run.png)
Мне очень не нравятся оба графика, но тот, который на основе вибраций, не нравится сильно больше. Поэтому решил продолжать историю с умной розеткой, а датчик вибраций отложить на потом. Впрочем, отключать пока его не стал на случай, если захочется еще разок покрутить крутилки и уменьшить количество фолзов, правда без особой надежды.

### WRS: final thoughts
Остались последние штрихи, т.к. графичек потребления это хорошо, но это еще не готовый источник событий для нотификаций. К счастью, в интернетах можно найти приличное количество автоматизаций подобного типа (замерили потребление -> приняли решение о состоянии).
  - Сам же я взял blueprint [Appliance has finished](https://github.com/metbril/home-assistant-blueprints/blob/748c87891712bc58b867a9af21c2450bebd8c0f2/automation/appliance_has_finished.yaml)  от metbril@: [appliance_has_finished.yaml](https://gist.github.com/buglloc/3ea4cf04a89e318fb58cc6d86886bbe0#file-appliance_has_finished-yaml)
  - Добавил его в Home Assistant и чуть поднастроил:
![](/assets/images/posts/wn/washer-automation.png)
  - Получил на выходе логику:
    * если 3 и более минуты потребление более 3W - машина начала работу
    * если же 5 и более минут потребление менее 3W - машина закончила работу
    * если машина закончила работу - стриггерить событие `washer_state`

Довольно незамысловато и ровно то, что нужно. На этом часть с определением "работает ли стиральная машина" я решил закончить и начать заниматься состоянием дверцы.

### Door state
Помните, я в самом начале говорил, что хорошо было бы знать, когда кто-то достал вещи и нет смысла топать на балкон? Вот этот раздел как раз об этом. Бонусом подумалось, что это же решение позволит еще больше снизить риск появления фолзов, т.к. мы можем увеличивать время простоя без ущерба для UX.

Учитывая, что машина у меня с вертикальной загрузкой - решение этой задачи пришло почти само собой:
  - берем какой-нить ESP32. У меня ждал своего часа [Super Mini ESP32-C3](https://a.aliexpress.com/_mtotKd0) (розовый, офк) за $3, который идеально вписался
  - лепим к нему датчик расстояния. Я прикупил на алике [VL53L0X](https://a.aliexpress.com/_mqz4mry) за $1
  - измеряя расстояние, принимаем решение, открыта ли дверца стиральной машины

Схематично, выглядит примерно так:
![](/assets/images/posts/wn/door-state.excalidraw.png)
Осталось дело за малым - подключаем `VL53L0X` к питанию и I²C шине `ESP32`:
```markdown
| ESP32          |  VL53L0X  |
|----------------|-----------|
|  3.3V          |    VIN    |
|  GND           |    GN     |
|  GPIO10        |    SCL    |
|  GPIO9         |    SDA    |
|  X (not used)  |    GPIO1  |
|  X (meh)       |    XSHUT  |
```

На `XSHUT` я забил, т.к. датчик у меня подключается только один и этот pin использоваться не будет. После небольшой пайки и клейки у меня получилась такая милаха:
![](/assets/images/posts/wn/tof-irl.jpg)

Осталось написать конфиг для [ESPHome](https://esphome.io/) ([сорец](https://gist.github.com/buglloc/3ea4cf04a89e318fb58cc6d86886bbe0#file-tof-esp-yaml)):
```yaml
esphome:
  name: washeresp
  friendly_name: WasherESP

esp32:
  board: esp32-c3-devkitm-1
  framework:
    type: arduino

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "DEADBEEF"

ota:
  password: "deadbeef"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  manual_ip:
    static_ip: 10.11.4.3
    gateway: 10.11.0.1
    subnet: 255.255.0.0
    dns1: 10.11.1.2

# I²C Bus необходим, т.к. к нему подключается vl53l0x
i2c:
  sda: GPIO9
  scl: GPIO10
  id: bus
  scan: true

# Не люблю много думать о времени, потому всегда использую Home Assistant Time Source
# doc: https://esphome.io/components/time/homeassistant.html
time:
  - platform: homeassistant
    id: homeassistant_time

# Status LED Light для того чтобы понимать текущее состояние
# doc: https://esphome.io/components/light/status_led
status_led:
  pin:
    number: GPIO8
    inverted: true

switch:
  # Restart Switch, чтобы мочь рестартить MCU из HA
  # doc: https://esphome.io/components/switch/restart.html
  - platform: restart
    name: Restart

binary_sensor:
  # Status Binary Sensor, дабы отрисовывать аптайм
  # doc: https://esphome.io/components/binary_sensor/status.html
  - platform: status
    name: Status

  # С помощью template перенесем логику обнаружения непосредственно на ESP
  # doc: https://esphome.io/components/binary_sensor/template.html
  - platform: template
    id: washer_window_opened
    publish_initial_state: true
    name: "Washer Window"
    device_class: window

sensor:
  # WiFi Signal Sensor, дабы лучше понимать что с WiFi
  # doc: https://esphome.io/components/sensor/wifi_signal.html
  - platform: wifi_signal
    name: "WiFi Signal Strength"
    update_interval: 60s

  # VL53L0X Time Of Flight Distance Sensor, собсно ради него весь и сыр-бор
  # doc: https://esphome.io/components/sensor/vl53l0x
  - platform: vl53l0x
    name: "Washer Door Distance"
    id: washer_door_distance
    address: 0x29
    update_interval: 5s
    long_range: false
    unit_of_measurement: "m"
    internal: true
    filters:
      - lambda: !lambda |-
          // считаем дверцу открытой, если растояние до нее менее 20 см
          bool isOpened = x <= 0.20;

          // дальше меняем состояние бинарного сенсора "washer_window_opened"
          bool curState = id(washer_window_opened).state;
          if (isOpened != curState) {
            id(washer_window_opened).publish_state(isOpened);
          }

          return {};
```

Прошить и добавить в Home Assistant:
![](/assets/images/posts/wn/tof-in-ha.png)

Самое время проверить в бою:

![](/assets/images/posts/wn/door-state-irl.excalidraw.png){: .align-center}


Это был последний компонент пазла. Теперь самое время объединить все вместе во имя счастья пользователей :)

### Нотификашки
И так, к этому моменту я располагал:
  - кастомным событием типа `washer_state` от умной розетки, состояние которого может запаздывать на 5 минут
  - бинарным сенсором `washer_window_opened` от датчика расстояния о состоянии дверцы
  - часы [Ulanzi TC001](https://ut.buglloc.com/iot/ulanzi-awtrix/), на которые надо бы доставить нотификашку
  - телефоны, на которые надо бы доставить пушик

Звучит, как время доставать JSONклей. И так оно и есть, потому как для этого я написал flow для [Node-RED](https://nodered.org/) ([сорец]()):
![](/assets/images/posts/wn/node-red-notify-flow.png)

За кубиками я спрятал следующую логику работы:
  - у нас есть два источника событий: `washer_state` и `washer_window_opened` + их имитации для ручного запуска в целях отладки
  - когда наступает `washer_state` (машина отработала по данным от розетки), то:
    * проверяем не было ли недавно события об открытии дверцы
    * если было - считаем за фолз или отставание (тип уже достали вещи)
    * если не было - отправляем в телефон и часики нотификашку
   - когда наступает `washer_window_opened`  (был открыта дверца), то:
     * записываем время наступления события, дабы учесть его при обработке `washer_state`
     * отменяем нотификашку на часах

Да, нотификации на часах получились не многопоточные, но я [пока] и не планировал добавлять новых источников :) Пробуем в деле:
<p><iframe width="560" height="315" src="https://www.youtube.com/embed/XCrweif3As8?si=4oKOQMWB01iAAzgJ" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe></p>

Что и требовалось:
  - сначала вручную стриггерил событие `washer_state` так, как это сделала бы розетка
  - прилетела нотификашка на телефон и часики
  - потом вручную стриггерил событие `washer_window_opened` так, как это сделал бы датчик расстояния
  - нотификашка пропала с часов
  - затем еще пару раз стриггерил событие `washer_state`, чтобы убедиться, что кеширование состояния дверцы работает и события будут отброшены как фолзовые
  - Profit!!11

### Closure
Это был довольно мучительный проект. Нет, не потому что сложный, а потому что эксперименты со стиральной машиной отнимают слишком много времени :(

Но, надеюсь, пользователи будут довольны, и эти усилия себя окупят ;)