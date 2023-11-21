---
title: "Sonoff S31: Flashing ESPHome"
classes: wide
header:
  og_image: /assets/images/posts/sonoff-s31/sonoff_cover.png
categories:
  - iot
---
Одним вечерним вечером мне понадобились умные розетки для всякой мелочевки. Ну знаете, зашедулить включение фумигатора от комариков и все в таком духе. Имеющийся [SMATRUL WiFi Socket](https://www.lazada.co.th/products/smatrul-20a16a-tuyasmart-life-wifi-socket-universal-us-eu-smart-plug-adapter-power-monitor-european-plug-wireless-remote-voice-timer-for-google-homealexatmall-genie-i2911432612-s10659100651.html?dsource=share&laz_share_info=674305705_9_100_100322224233_674305706_null&laz_token=88cc5f36711df9024f45cb3f493e2031&exlaz=e_iqeJxmuQOjPGip8qo24MCSTeQCjDld4siVQ4kY1ODqrCXvTKdNAtBb55%2BgIiSkmNnXkPoNuoMIIpR8IEI%2FLi1U0sC6tUOx2TD3WnASIas1Y%3D&sub_aff_id=social_share&sub_id2=674305705&sub_id3=100322224233&sub_id6=CPI_EXLAZ) из поста [про стиралочку](https://ut.buglloc.com/iot/washer-notifier/) мне не понравился, поэтому я занырнул в поиск Lazada/AliExpress и вынырнул с [Sonoff S31](https://sonoff.tech/product/smart-plugs/s31-s31lite/). Собсно, начало коротенькой зарисовочки о ее перепрошивке вы сейчас и читаете ;)
![](/assets/images/posts/sonoff-s31/sonoff_cover_tgdraw.png){: .align-center}

# О Sonoff S31

![](/assets/images/posts/sonoff-s31/sonoff_wall.png){: .align-center}
Для начала, как и положено, пара слов о выборе:
  - Стоит как средний бургер с колой, т.е. около $5
  - Type B socket (вариация US) - довольно популярная тема в Таиланде
  - Умеет измерять потребление энергии. Кстати, есть и Sonoff S31 Lite без
  - До 15A нагрузки
  - [ESP8266](https://www.espressif.com/en/products/socs/esp8266) вместо сердца
  - Красивенькая, но разбирается отверткой

Будь вместо Type B что-то универсальное, вообще цены бы не было. Но и так не дурно. Особенно учитывая, что ESP8266 + разборка отверткой = привет [ESPHome](https://devices.esphome.io/devices/Sonoff-S31). 

# Разбираем
Нам понадобится отвертка и что-то для защелок (я использую пластиковую штуку для разборки телефонов). Прямые руки, кстати, совсем не обязательны - я проверял ;)

  - Начинаем с того, что снимаем серенькую крышку:
![](/assets/images/posts/sonoff-s31/sonoff_diss_0.png){: .align-center}

  - Добираемся до шурупов и откручиваем их:
![](/assets/images/posts/sonoff-s31/sonoff_diss_1.png){: .align-center}

  - Вуаля, вот они нужные пины:
![](/assets/images/posts/sonoff-s31/sonoff_diss_done.png){: .align-center}

# Прошиваем
Для прошивки все еще не нужны прямые руки, но нужен паяльник и USB to Serial converter (у меня какой-то дефолтный на базе FT232 с алика).

  - Припаиваемся к пинам VCC, RX, TX и GND. К счастью, аккуратность тут не важна:
![](/assets/images/posts/sonoff-s31/flash_0.jpg){: .align-center}

  - Подключаемся к serial converter:
```markdown
| USB2Serial  |   S31   |
|-------------|---------|
|  VCC (3.3V) |   VCC   |
|    GND      |   GND   |
|    RX       |   TX    |
|    TX       |   RX    |
```

  - Зажимаем кнопку на платке (подключена к `GPIO0`) для перевода ESP8266 в режим бутлоадера и втыкаем USB to Serial в комп
  - Заводим новое устройство в ESPHome:
![](/assets/images/posts/sonoff-s31/flash_1.jpg){: .align-center}
  - Дожидаемся окончания процесса и ребутаем S31
  - Хоба, готово! Можем подключиться по воздуху и почитать лог:
![](/assets/images/posts/sonoff-s31/flash_2.jpg){: .align-center}

На этом с прошивкой ESPHome закончили, мы молодцы :) Все дальнейшие манипуляции можно сделать по воздуху, поэтому собираем нашу розетку обратно.
# Последние шаги
ESPHome и OTA это, конечно, хорошо, но у нас тут вообще-то розетка! Что ж, шьем ей конфиг (взят из документации [Sonoff S31 для ESPHome](https://devices.esphome.io/devices/Sonoff-S31)):
```yaml
esphome:
  name: dinners31plug
  friendly_name: DinnerS31Plug

esp8266:
  board: esp01_1m

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "DEADBEEF"

ota:
  password: "DEADBEEF"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  manual_ip:
    static_ip: 10.11.5.1
    gateway: 10.11.0.1
    subnet: 255.255.0.0
    dns1: 10.11.1.2

time:
  - platform: homeassistant
    id: homeassistant_time

uart:
  rx_pin: RX
  baud_rate: 4800

binary_sensor:
  - platform: gpio
    pin:
      number: GPIO0
      mode: INPUT_PULLUP
      inverted: True
    name: "Sonoff S31 Button"
    on_press:
      - switch.toggle: relay
  - platform: status
    name: "Sonoff S31 Status"

sensor:
  - platform: wifi_signal
    name: "Sonoff S31 WiFi Signal"
    update_interval: 60s
  - platform: cse7766
    current:
      name: "Sonoff S31 Current"
      accuracy_decimals: 1
    voltage:
      name: "Sonoff S31 Voltage"
      accuracy_decimals: 1
    power:
      name: "Sonoff S31 Power"
      accuracy_decimals: 1
      id: my_power
  - platform: total_daily_energy
    name: "Sonoff S31 Daily Energy"
    power_id: my_power

switch:
  - platform: gpio
    name: "Sonoff S31 Relay"
    pin: GPIO12
    id: relay
    restore_mode: ALWAYS_ON

status_led:
  pin:
    number: GPIO13
    inverted: True
```

Добавляем нашу `DinnerS31Plug` в Home Assistant (воткнул в нее что-то, дабы проверить power monitoring):
![](/assets/images/posts/sonoff-s31/sonoff_ha.png){: .align-center}

Собсно, чего мы и добивались! Дальше можно мутить нужные нам автоматизации, но об этом как-то в другой раз при хорошем случае. А пока всем кота (づ˶•༝•˶)づ♡
