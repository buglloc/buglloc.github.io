---
title: "Homemade FDE: BoundBoxESP - Part One"
classes: wide
header:
  og_image: /assets/images/posts/hmfde/bound-box-principal.excalidraw.png
categories:
  - home-infra
---
Если прошлый пост (см. [Homemade FDE: Part Zero](https://ut.buglloc.com/home-infra/hmfde-part-zero/)) в большей степени был вступительным, то в этом я постараюсь максимум внимания уделить аппаратно-программной реализации. Ну и в целом рассказать про [BoundBoxESP](https://github.com/buglloc/BoundBoxESP) :)

**КДПВ**

На мой вкус, в этой части собрано максимальное количество боли и унижений, поэтому рассчитываю, как минимум, развлечь ^^.

## База
Но сначала напомню концепцию:
  - некий утюг идет по SSH в наш сервис
  - отправляет свой секрет
  - в ответ получает новый секрет, как результат работы key derivation function (KDF)
  - использует его для своих утюжных делишек (FDE, unwrapping, etc)

Таким образом, в покое, ни один из участников не обладает финальным секретом и нуждаются в друг дружке. Визуально выглядит как-то так:

![bound-box-draft](/assets/images/posts/hmfde/kdf-overview.excalidraw.png)

## Дела хардварные
Невероятно, но, с учетом нагрузки на будущий сервис, его можно крутить хоть на картошке :) Вариантов по хранению/получению мастер-ключа у нас тоже довольно много. А значит наиболее честный ответ в выборе железа будет - я так захотел.

Например, это могло бы быть что-то с TPM. Будь-то amd64 или arm64:
<figure style="width: 800px">
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/posts/hmfde/hw-variants-tpm.jpg" alt="">
  <figcaption class="align-center">ZX01 Plus / RPi4 + TPM9670</figcaption>
</figure>
И, на мой вкус, это было бы чертовски скучно. А мы тут вообще-то приятное с полезным совмещаем, не до скучных решений знаете ли. Из чуть более приземленных соображений - очень плохая утилизация, тот же N100 в ZX01 будет во-о-бще ничем не занят примерно все время, захочется или селить что-то рядом (wat?) или как-то совсем глупо получается.

Чуть разбавить скуку и выровнять КПД можно было бы с помощью одноплатников форм-фактора Raspberry Pi Zero (или Orange Pi Zero) со смарт-картой (особенно MangoPi MQ-PRO на RISC-V, хе-хе):
<figure style="width: 800px">
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/posts/hmfde/hw-variants-piv.jpg" alt="">
  <figcaption  class="align-center">Radxa Zero + YubiKey 5C Nano / RPi Zero + Touch E-Ink Display / MangoPi MQ-PRO + YubiKey 5C Nano</figcaption>
</figure>
Но, будем честны, веселее становится в основном за счет проблем со скоростью загрузки линуксов. Это решабельно, но я сказал, что хочу больше веселья! :)

И тут с ноги врываются разнообразные MCU. Выбор там, к счастью, довольно большой (e.g. nRF52840, ESP32, RP2040, etc), но думаю ESP32 с пин-падом или сенсорным экраном будут отличным стартом. Вон каких красавчиков наалишил:
<div>
<figure style="width: 200px" class="align-left">
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/posts/hmfde/LILYGO-LNURLPoS.png" alt="">
  <figcaption><center><a href="https://www.aliexpress.com/item/1005003589706292.html" target="_blank">LILYGO LNURLPoS</a></center></figcaption>
</figure>
<figure style="width: 200px" class="align-left">
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/posts/hmfde/T-Display-S3-Pro.png" alt="">
	  <figcaption><center><a href="https://www.aliexpress.com/item/1005006199048187.html" target="_blank">LILYGO T-Display-S3-Pro</a></center></figcaption>
</figure>
<figure style="width: 200px" class="align-left">
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/posts/hmfde/M5Knob.png" alt="">
	  <figcaption><center><a href="https://www.aliexpress.com/item/1005006102334456.html" target="_blank">M5Stack Smart Knob</a></center></figcaption>
</figure>

<div class="cf"></div>
</div>

Так я и поступил, выбрав для текущей версии [BoundBoxESP](https://github.com/buglloc/BoundBoxESP#TODO-ЗАПИНИТЬ-ВЕРСИЮ) :
  - [LILYGO T-Display S3 AMOLED](https://www.aliexpress.com/item/1005005416973021.html)  как основу всего + средство ввода
  - [3.7V 1000mAh 503450 Battery](https://www.aliexpress.com/item/4000151489952.html) дабы переживать электриков
  - [W5500 Ethernet Module](https://www.aliexpress.com/item/1005006127400904.html) потому что лучший WiFi - проводной WiFi

Доску [LILYGO T-Display S3 AMOLED](https://www.lilygo.cc/products/t-display-s3-amoled) я выбрал по нескольким причинам:
  1. Чертовски миленькая :)
  2. [T-Display-S3-Pro](https://www.lilygo.cc/products/t-display-s3-pro) на момент старта проекта не был широко доступен
  3. Выведено достаточное количество хороших пинов для подключения W5500 по SPI

Ну и в целом это добротная, современная доска с двухъядерным ESP32-S3R8, 16MB Flash, 8MB PSRAM, встроенной зарядкой для аккумулятора и 1.91" сенсорным (CST816) IPS Amoled экраном (RM67162) разрешением 240x536 точек. Из минусов только отсутствие вывода RST пина и какого-либо крепежа (ушки там для M2.5, например), но чем-то пришлось пожертвовать. Помимо прочего, ESP32-S3 обещал мне какой-никакой [Secure Boot](https://docs.espressif.com/projects/esp-idf/en/v5.1.2/esp32s3/security/secure-boot-v2.html), [Flash](https://docs.espressif.com/projects/esp-idf/en/v5.1.2/esp32s3/security/flash-encryption.html) и [NVS](https://docs.espressif.com/projects/esp-idf/en/v5.1.2/esp32s3/api-reference/storage/nvs_flash.html#nvs-encryption) encryption, что само по себе довольно не плохо, хоть, конечно, и не так ценно при вводе пароля. Надеюсь не сплоховал :)

## Дела паяльные
К счастью, подключать нам только W5500:

```md
|  T-Display  |     W5500      |
|-------------|----------------|
|    3v3      |   3v3          |
|    GND      |   GND          |
|    GPIO10   |   SPI CS/SS    |
|    GPIO11   |   SPI MOSI     |
|    GPIO12   |   SPI SCK      |
|    GPIO13   |   SPI MISO     |
|    GPIO14   |   INT          |
|    GPIO15   |   RST          |
```

Вариант для визуалов:
![bound-box-schematic](/assets/images/posts/hmfde/boundbox-schematic.excalidraw.png)

Запаиваем и проверяем на весу, что все выглядит рабочим:
![bound-box-schematic](/assets/images/posts/hmfde/boundbox-air-test.jpg)

И "корпусируем":

**ФОТО**

Та-да!

**ФОТО**

Фуф, можем откладывать в сторону паяльник и DIY набор для детей от 5 до 10 лет, они нам больше не понадобятся. Далее ручной труд, только в привычном для нас смысле ;)
## Дела программные
Написан  [BoundBoxESP](https://github.com/buglloc/BoundBoxESP) на C++ с использованием [Espressif IoT Development Framework (ESP-IDF)](https://idf.espressif.com/) и [LibSSH-ESP32](https://github.com/ewpa/LibSSH-ESP32).

### Почему ESP-IDF?
Почему ESP-IDF, а не Arduino спросите вы? А потому что ESP-IDF:
  - предлагает писать на C++23 (см. [C++ Support](https://docs.espressif.com/projects/esp-idf/en/v5.1.2/esp32/api-guides/cplusplus.html)) или C, а не каком-то подмножестве C++ именуемым Arduino "language"
  - использует запчасти из FreeRTOS (см. [FreeRTOS (ESP-IDF)](https://docs.espressif.com/projects/esp-idf/en/v5.1.2/esp32/api-reference/system/freertos_idf.html)) с шедулером задач, событийной моделью и прочими благами цивилизации. Больше никаких `if (curMillis - prevMillis >= interval)` на ровном месте (e.g. [Multitasking with Arduino](https://www.seeedstudio.com/blog/2021/05/11/multitasking-with-arduino-millis-rtos-more/))
  * что логично, своевременно получает поддержку новых Espressif SoC. Так, например, поддержка ESP32 С6/H2 все еще не докатилась через три слоя абстракций (в норме это [ESP-IDF](https://github.com/espressif/esp-idf) -> [Arduino-ESP32](https://github.com/espressif/arduino-esp32/) -> [PlatformIO](https://platformio.org/)).
  * ощущается целостным фреймворком, со своей идеологией и базовыми принципами. Arduino же просто сборище бибок, которых связывает страсть к `Begin` и `Loop`
  * имеет [шикарную документацию](https://docs.espressif.com/projects/esp-idf/en/v5.1.2/esp32s3/get-started/index.html), большая часть разделов которой сопровождается [полноценными примерами](https://github.com/espressif/esp-idf/tree/master/examples)

Конечно, у Arduino тоже есть свои плюсы и было бы странно их отрицать:
  * кроссплатформенность и статус дефолтного фреймворка для многих MCU
  - отсюда несравнимо большее комьюнити
  - отсюда 1st class support разными производителями

Да что уж там, первая версия BoundBoxESP была написана именно с его использованием (см. [v0.0.1](https://github.com/buglloc/BoundBoxESP/tree/v0.0.1)). LilyGo решили, что за пределами Arduino жизни нет (см. [LilyGo-AMOLED-Series](https://github.com/Xinyuan-LilyGO/LilyGo-AMOLED-Series)), а потому и я решил, что обману судьбу, обогну все грабли и за пару утро-ночеров закончу. План был отличный, пока я не понял что:
  -  [LibSSH-ESP32](https://github.com/ewpa/LibSSH-ESP32) и ардуиновская [Ethernet](https://github.com/arduino-libraries/Ethernet) используют не совместимый сетевой стэк, что требует каких-то не тривиальных приседаний
  - если альтернативный варик - заиспользовать [ETH](https://github.com/espressif/arduino-esp32/tree/master/libraries/Ethernet) из Arduino-ESP32, но поддержка SPI Ethernet была сделана/починена только в версии [3.0.0 Alpha](https://github.com/espressif/arduino-esp32/pull/8712)
  - а посколько v3.0.0 это альфа, поддержи в PlatformIO мы до релиза [не увидим](https://github.com/platformio/platform-espressif32/issues/1211)
  - господи, как же я обожаю это дерьмо!

Пожалуй, если бы мне ардуиновская экосистема была близка, то я бы ченить придумал. Но это не так, плюс я давно хотел попробовать ESP-IDF. Потому потратив еще пару утро-ночеров портировал поддержку периферии [LilyGo-AMOLED](https://github.com/buglloc/BoundBoxESP/tree/2a3f212581254fd31d072d35ea22b18fabaf1f16/components/t_amoled) под ESP-IDF, а затем и весь проект целиком (см. [v0.0.2](https://github.com/buglloc/BoundBoxESP/tree/v0.0.2))

### Почему LibSSH-ESP32?
Я знаю две рабочие реализации SSH под ESP32 - [LibSSH-ESP32](https://github.com/ewpa/LibSSH-ESP32) (порт [libssh](https://www.libssh.org/)), который я использовал в не законченном Escrow сервисе [Lupa-ESP32](https://github.com/buglloc/Lupa-ESP32) и [WolfSSH](https://github.com/wolfSSL/wolfssh). Сначала я взял [LibSSH-ESP32](https://github.com/ewpa/LibSSH-ESP32), т.к. он был мне знаком + без проблем заводился под Arduino. Но с переездом на ESP-IDF решил таки попробовать и [WolfSSH](https://github.com/wolfSSL/wolfssh), вдруг годнота? Сказано - сделано, утром ли ночью ли родился BoundBoxESP [v0.0.2](https://github.com/buglloc/BoundBoxESP/tree/v0.0.2), работающий поверх WolfSSH. И тут я снова  поймал дизмораль :( Дело в том, что помимо неполноценности (e.g. нормальная поддержка ED25519 вот-вот докатывается), он еще и был чертовски медленным. Может я не сумел приготовить поддержку hardware encryption в WolfSSL, но вот две реализации, работающие по ethernet:
  - поверх WolfSSH:
```bash
$ time bash -c "echo '{}' | ssh tst@192.168.2.138 -- /help > /dev/null"
0.03s user 0.01s system 1% cpu 2.606 total
```
  - и LibSSH:
```bash
$ time bash -c "echo '{}' | ssh tst@192.168.2.138 -- /help > /dev/null"
0.03s user 0.01s system 9% cpu 0.399 total
```

Думаю тут комментариев не нужно. Наверное, будь у WolfSSH более лучшая или полноценная реализация ~~Open~~SSH я бы ченить придумал. Но раз нет - то я вернулся к LibSSH-ESP32.
### BoundBoxESP v1.0.0-alpha
Общая логика текущей предрелизной версии выглядит так:
![bound-box-schematic](/assets/images/posts/hmfde/boundbox-esp-logic.excalidraw.png)Так мы:
  1. Инициализируем экран, сеть и прочую периферию
  2. Проверяем наличие прикопанных в [NVS](https://docs.espressif.com/projects/esp-idf/en/v5.1.2/esp32s3/api-reference/storage/nvs_flash.html) секретов (сейчас это SSH ключ и зигота мастер-ключа). В случае отсутствия - генерируем новые
  3. Дожидаемся подключения по сети (возможно зря, не решил пока)
  4. Запрашиваем PIN и получаем мастер-ключ: `HMAC-SHA-256(NVSKey, EnteredPIN)`
  5. Генерируем тестовый секрет: `HMAC-SHA-256(MasterKey, VerificationSalt)`
  6. Спрашиваем ОК ли он
  7. Если нет, то возвращаемся к п4 и снова запрашиваем PIN
  8. Если все норм, то:
  - стартуем SSHD
  - обрабатываем входящие команды
  - показываем подписочные нотификашки при необходимости
  - работаем, в общем :)

Благодаря такой логике мы смогли получить три важных свойства:
  - Прикованность мастер-ключа к конкретному устройству, т.к. один и тот же PIN на разных устройствах (с разными NVSKey) породит разные мастер-ключи
  - Но при этом резервируемость, благодаря хранению секретов в *зашифрованном* NVS. Т.е. в случае беды мы можем налить новое устройство с такими же секретами и получить полноценную замену
  - И антибрутальность, т.к. с т.з. стороннего наблюдателя любая пара PIN + NVSKey валидны. Да, тут чуть пострадал UX, зато не нужно хранить счетчик попыток ввода на флешке, дропать секреты при превышении и вот это все

Вот, кстати, примеры командочек для админа:
```bash
$ ssh -l buglloc bbw0.buglloc.cc -- /help | jq .commands

[
  {
    "command": "/hmac/secret",
    "description": "Generate HMAC secret for req[\"salt\"] into rsp[\"secret\"]"
  },
  {
    "command": "/help",
    "description": "Returns commands info"
  },
  {
    "command": "/status",
    "description": "Returns BoundBoxESP status"
  },
  {
    "command": "/restart",
    "description": "Restart BoundBoxESP in req[\"delay_ms\"]"
  },
  {
    "command": "/secrets/store",
    "description": "Store runtime secrets from req[\"secrets\"]"
  },
  {
    "command": "/secrets/reset",
    "description": "Reset secrets to it's default values"
  }
]
```

А вот обычного работяги:
```bash
$ ssh bbw0.buglloc.cc \
  -l 'SHA256:C0Q14mSJLVITEyGsP6QLE1Z/GfTwEq1mLzVemnVch0E' \
  -- /help | jq .commands

[
  {
    "command": "/hmac/secret",
    "description": "Generate HMAC secret for req[\"salt\"] into rsp[\"secret\"]"
  },
  {
    "command": "/help",
    "description": "Returns commands info"
  }
]
```

И самое важное - получение секретика (см. [концепцию](#база)):
```bash
$ echo '{"salt":"cHhrQThGeFdUWDZCMWg2MVFLQTBONEpXCg=="}' | ssh bbw0.buglloc.cc \
  -l 'SHA256:C0Q14mSJLVITEyGsP6QLE1Z/GfTwEq1mLzVemnVch0E' \
  -- /hmac/secret | jq .

{
  "secret": "O+6rFW2UDigIXbzpKUpiigokXu/JeAZXZY8p5Xzm8ZA=",
  "ok": true,
  "id": "817682-8c926b"
}
```

## Демо
**TODO**

## Final thoughts
**TODO**

Всем кота (づ˶•༝•˶)づ♡