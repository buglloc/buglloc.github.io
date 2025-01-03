---
title: "Ulanzi TC001 Desktop Clock"
header:
  og_image: "/assets/images/posts/awtrix-aweeting-upcoming.gif"
classes: wide
categories:
  - iot
redirect_from:
  - /iot/ulanzi-awtrix/
---

До сегодня я считал "умные" настольные часы не многим полезнее "умного" чайника. Но [Ulanzi TC001 Desktop Clock](https://www.ulanzi.com/products/ulanzi-pixel-smart-clock-2882?ref=28e02dxl) + [Awtrix Light]( https://github.com/Blueforcer/awtrix-light) смогли меня разубедить. Кто бы мог подумать, но иметь какую-то информацию всегда на виду и правда довольно удобно :)

Текущий результат:
{% include video id="1042399383" provider="vimeo" %}

### От желания
Как-то я насмотрелся блогеров и решил попробовать примерить на себя какой-нить пиксельный настольный дисплейчик. Например, ченить в духе [Divoom](https://divoom.com/products/pixoo-64). Собравшись с мыслями, я выписал для себя незамысловатый набор критериев:
  - что-то недорогое, дабы проверить концепцию без заемных средств
  - что-то с открытой прошивкой или хотя бы офф. API. Для любителей IOT это не новость, но производители очень не любят продавать вам железо, а то вы еще посчитаете его своим, хех
  - не выглядеть колхозом, т.к. это гостиная

А, ну и я не хотел паять сам. Я знаю, что взять ESP32/RP2040/etc и приговнячить к нему матрицу или IPS/E-Ink дисплей дело не хитрое, но...вы бы видели синусоиду моих рук, ставить сие творение в гостиной я бы не рискнул  :)

### К выбору решения
После недолгих поисков я наткнулся на настольные часики Ulanzi TC001 ([алик](https://a.aliexpress.com/_mMAJRY2)):
  - со скидосами обошлись мне в $36
  - с ESP32 вместо сердца
  - имеет usb-c порт, подключенный в serial interface (да-да, можно шить, не цепляясь к пинам программатором!)
  - для них существует отличная альтернативная прошивка с живым комунити: [Awtrix Light]( https://github.com/Blueforcer/awtrix-light) 
  - ряд не интересных мне фич, например, пищалка и батарейка. Но может кому-то будет важно :)

Вот так они выглядят в моей гостинной:
![ulanzi tv](/assets/images/posts/ulanzi-tv.jpg)

Делать детальный обзор и то, как он шьется, я не стану. Шьется он настолько просто, что справится и ПТУшник типа меня. А обзоры лучше пишут другие:
  - [This is the BEST MATRIX DISPLAY CLOCK for Home Assistant! by Smart Home Junkie](https://www.smarthomejunkie.net/this-is-the-best-matrix-display-clock-for-home-assistant/)
  - [Ulanzi TC001 Desktop Clock by DezeStijn](https://sequr.be/blog/2023/03/ulanzi-tc001-desktop-clock-awtrix/)
  - Just Google It %)

Скажу лишь, что в целом я остался доволен (особенно учитывая цену):
  - разрешение маловато, но в этом есть свой шарм
  - хорошее комунити, можете залетать в discord там есть всякая дичь для вдохновения
  - Humidity&&Temperature sensor полное дно. По идее там ченить из серии SHT3x/AHT2x, поэтому качество ожидаемо
  - MQTT/HTTP API-first реализация Awtrix Light позволяет управлять ими как пожелаешь и интегрировать с чем пожелаешь

### И опыту использования
С чего, как вы думаете, начинается использование? Правильно! С того, что надо бы вкорячить хоть какую-то аутентификацию/авторизацию (ну вы знаете...IOT...), чем [я и занялся](https://github.com/Blueforcer/awtrix-light/pull/268). Иначе любой школьник сможет или утащить ваши явки и пароли (wifi, mqtt, вот это все), или перешить часы во что пожелает. Не скажу, что это все проблемы с безопасностью, но без починки этого хз как продолжать.

Что ж, после добавления авторизации выставил часы в прокаженный VLAN, подключил к MQTT и добавил в Home Assistant.
Самое время начинать играть с кастомными приложеньками! Благо, за счет API-first подхода делать это одно удовольствие. Посмотрите в разделе ["Custom Apps and Notifications"](https://blueforcer.github.io/awtrix-light/#/api?id=custom-apps-and-notifications) документации сами. Да-да, все НАСТОЛЬКО просто:
  - пуляем JSONку в HTTP-ручку или MQTT-топик
  - часики показывают, что велено

Например, показать нотификацию:

Шаг 1: отправляем JSONку в HTTP-api
```
$ http -v -a "$AWTRIX_AUTH" http://awtrix.iot.lan/api/notify text='Oh, my' stack:=false icon=10205 duration=5
POST /api/notify HTTP/1.1
Accept: application/json, */*;q=0.5
Accept-Encoding: gzip, deflate
Authorization: Basic XXXXXXXXX
Connection: keep-alive
Content-Length: 68
Content-Type: application/json
Host: awtrix.iot.lan
User-Agent: HTTPie/3.2.2

{
    "duration": "5",
    "icon": "10205",
    "stack": false,
    "text": "Oh, my"
}


HTTP/1.1 200 OK
Access-Control-Allow-Headers: *
Access-Control-Allow-Methods: *
Access-Control-Allow-Origin: *
Connection: close
Content-Length: 2
Content-Type: text/plain

OK
```


Шаг 2: любуемся!
<div style="padding: 10px;background: #aaa;"><img src="/assets/images/posts/awtrix-notify-demo.gif"/></div>

Признаться, делать каких-то супер интерактивностей я пока не стал (например, нотификацию от стиралки или ченить в таком духе), но занялся информационными приложеньками:
  - курс валют, который меня нынче беспокоит больше обычного :)
  - информацию о рабочих встречах

Как и положено, решил я их выполнить в двух вариациях, дабы было с чем сравнить :) О чем ниже.
### Приложенька: курс валют
С недавних пор курс валют меня довольно сильно беспокоит, поэтому прикольно иногда цепляться взглядом за, то каков нынче тренд и не пора ли предпринять каких-то действий. Штырить в него 24/7 смысла нет, а вот случайно обращать внимание нормис.

Курс валют я делал в [Node-RED](https://nodered.org/) и двух вариациях:
  - RUB -> USDT && USDT -> THB в P2P Binance, пока он еще работал ([source](https://gist.github.com/buglloc/a93405ea786dfa4add584bc926406ffe)):
![binance flow](/assets/images/posts/awtrix-binance-flow.png)
  * RUB -> THB в Contant:
![contant flow](/assets/images/posts/awtrix-contact-flow.png)

Работает, как вы понимаете, не замысловато:
 - по таймеру или по кнопке (физической или виртуальной) стартуем flow
 - дергаем HTTP-ручку с данными
 - парсим&&считаем
 - пушим в mqtt для отображения

В целом, как клей [Node-RED](https://nodered.org/) мне скорее понравился. Делать в нем какую-либо логику я бы не стал, а вот переложить JSON'ку самое то.

### Приложенька: Aweeting
Вторая, важная для меня приложенька - отображение грядущей/текущей рабочей встречи. Ну вы знаете, чтобы домашние не таранили дверь кабинета во время ;)
Как и курс валют большой хитростью не обладает, но логики имеет больше, а потому выполнена в качеcтве standalone приложеньки: [Aweeting](https://github.com/buglloc/aweeting)

Технически вся логика сводится к простым действиям:
  - парсим рабочий календарь (у меня это ical доступный по http)
  - объединяем встречи в промежутки (на случай встреч идущих друг за дружкой)
  - пушим информацию о грядущей/текущей встрече

Что-то похожее я раньше делал для напоминания о встречах Алисой, пригодились наброски :)

Вот как это выглядит:
  - встреча начнется через 13 минут:
<div style="padding: 10px;background: #aaa;"><img src="/assets/images/posts/awtrix-aweeting-upcoming.gif"/></div>
  - встреча закончится через час:
<div style="padding: 10px;background: #aaa;"><img src="/assets/images/posts/awtrix-aweeting-on-air.gif"/></div>

В следующий раз, скорее всего, я бы отказался от прямых походов из приложения в MQTT. Вместо этого стоило пойти другим путем:
  - зафигачить HTTP-api
  - ходить в него из Node-RED

Было бы и гибче, и прямее, кмк :)

### Closure
Как я и говорил в самом начале, сам формат мне скорее зашел. Думаю в будущем попробовать применить нотификашки (от той же стиралки), ну и подумать не докупить ли/собрать ли еще какого-нить схожего железа :)
