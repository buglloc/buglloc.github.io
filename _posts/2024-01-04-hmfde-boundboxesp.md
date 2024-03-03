---
title: "Homemade FDE: BoundBoxESP"
classes: wide
header:
  og_image: /assets/images/posts/hmfde/part-one-cover.jpg
categories:
  - home-infra
---
Если прошлый пост (см. [Homemade FDE: Part Zero](https://ut.buglloc.com/home-infra/hmfde-part-zero/)) в большей степени был вступительным, то в этом я хочу рассказать про [BoundBoxESP](https://github.com/buglloc/BoundBoxESP), как о всратом сердце всей истории с домашним шифрованием :)
<figure style="max-width: 800px">
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/posts/hmfde/part-one-cover.jpg" alt="">
</figure>

## База
Но сначала напомню концепцию:
  - некий утюг идет по SSH в наш сервис
  - отправляет свой секрет
  - в ответ получает новый секрет, как результат работы KDF
  - использует его для своих утюжных делишек (FDE, unwrapping, etc)

Таким образом, в покое ни один из участников не обладает финальным секретом и нуждается в друг дружке. Визуализация:

![bound-box-draft](/assets/images/posts/hmfde/kdf-overview.excalidraw.png)

## Дела хардварные
Невероятно, но с учетом нагрузки на будущий сервис его можно крутить на любой около-современной картошке :) Вариантов по хранению/получению мастер-ключа у нас тоже довольно много. А значит наиболее честный ответ в выборе железа будет "я так захотел".

Например, это могло бы быть что-то с TPM. Будь-то amd64 или arm64:
<figure style="max-width: 600px">
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/posts/hmfde/hw-variants-tpm.jpg" alt="">
  <figcaption class="align-center">ZX01 Plus / RPi4 + TPM9670</figcaption>
</figure>
И, на мой вкус, это было бы чертовски скучно. А мы тут вообще-то приятное с полезным совмещаем, не до скучных решений знаете ли. Из чуть более приземленных соображений - очень плохая утилизация. Тот же N100 в ZX01 будет во-о-бще ничем не занят примерно все время, захочется или селить что-то рядом (wat?), или как-то совсем глупо получается.

Чуть разбавить скуку и выровнять КПД, можно было бы с помощью одноплатников форм-фактора Raspberry Pi Zero (или Orange Pi Zero) со смарт-картой:
<figure style="max-width: 600px">
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/posts/hmfde/hw-variants-piv.jpg" alt="">
  <figcaption  class="align-center">Radxa Zero + YubiKey / RPi Zero + Touch E-Ink / MangoPi MQ-PRO + YubiKey</figcaption>
</figure>
Но, будем честны, веселее становится в основном за счет проблем со скоростью загрузки линуксов. Это решабельно, но я сказал, что хочу больше веселья! :)

И тут в тред врываются разнообразные MCU. Выбор там, к счастью, довольно большой (e.g. nRF52840, ESP32, RP2040, etc), но думаю ESP32 с пин-падом или сенсорным экраном будут отличным стартом. Вон каких красавчиков наалишил:
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
  - [3.7V 1000mAh 503450 Battery](https://www.aliexpress.com/item/4000151489952.html) как ответ электрику
  - [W5500 Ethernet Module](https://www.aliexpress.com/item/1005006127400904.html), потому что лучший WiFi - проводной WiFi (но это не точно)

Полный комплект обошелся мне в $24.22 + $5.81 + $2.56 = $32.59 или половинка YubiKey 5C Nano.

Если интересно, то доску [LILYGO T-Display S3 AMOLED](https://www.lilygo.cc/products/t-display-s3-amoled) я выбрал по нескольким причинам:
  1. Чертовски миленькая :)
  2. [T-Display-S3-Pro](https://www.lilygo.cc/products/t-display-s3-pro) на момент старта проекта не был широко доступен
  3. Выведено достаточное количество хороших пинов для подключения W5500 по SPI

Ну и в целом это добротная, современная доска на базе ESP32-S3R8 с двухъядерным Xtensa® LX7, 16MB Flash, 8MB PSRAM, встроенной зарядкой для аккумулятора и 1.91" сенсорным (CST816) IPS Amoled экраном (RM67162) разрешением 240x536 точек. Из минусов - только отсутствие вывода RST пина и какого-либо крепежа (ушки бы там для M2.5, например), но чем-то пришлось пожертвовать. Помимо прочего, ESP32-S3 обещал мне какой-никакой [Secure Boot](https://docs.espressif.com/projects/esp-idf/en/v5.1.2/esp32s3/security/secure-boot-v2.html), [Flash](https://docs.espressif.com/projects/esp-idf/en/v5.1.2/esp32s3/security/flash-encryption.html) и [NVS](https://docs.espressif.com/projects/esp-idf/en/v5.1.2/esp32s3/api-reference/storage/nvs_flash.html#nvs-encryption) encryption, что само по себе довольно не плохо, хоть, конечно, и не так ценно при вводе пароля. Надеюсь, не сплоховал :)

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

![air-test](/assets/images/posts/hmfde/boundbox-air-test.jpg)

"Корпусируем":

![underwear](/assets/images/posts/hmfde/boundbox-underwear.jpg)

Та-да!

![final](/assets/images/posts/hmfde/boundbox-final.jpg)


Фуф, можем откладывать в сторону паяльник и DIY набор для детей от 5 до 10 лет, они нам больше не понадобятся. Далее ручной труд, только в привычном для нас смысле ;)
## Дела программные
Написан  [BoundBoxESP](https://github.com/buglloc/BoundBoxESP) на C++ поверх [ESP-IDF](https://idf.espressif.com/) с использованием [LibSSH-ESP32](https://github.com/ewpa/LibSSH-ESP32) и [LVGL](https://lvgl.io/). Почему ESP-IDF, а не Arduino спросите вы? А потому что ESP-IDF:
  - предлагает писать на C++23 (см. [C++ Support](https://docs.espressif.com/projects/esp-idf/en/v5.1.2/esp32/api-guides/cplusplus.html)) или C, а не каком-то подмножестве C++ именуемым Arduino "language"
  - использует запчасти из FreeRTOS (см. [FreeRTOS (ESP-IDF)](https://docs.espressif.com/projects/esp-idf/en/v5.1.2/esp32/api-reference/system/freertos_idf.html)) с шедулером задач, событийной моделью и прочими благами цивилизации. Больше никаких `if (curMillis - prevMillis >= interval)` на ровном месте (e.g. [Multitasking with Arduino](https://www.seeedstudio.com/blog/2021/05/11/multitasking-with-arduino-millis-rtos-more/))
  * что логично, своевременно получает поддержку новых Espressif SoC. Так, например, поддержка ESP32 С6/H2 все еще не докатилась через три слоя абстракции (в норме это [ESP-IDF](https://github.com/espressif/esp-idf) -> [Arduino-ESP32](https://github.com/espressif/arduino-esp32/) -> [PlatformIO](https://platformio.org/)).
  * ощущается целостным фреймворком, со своей идеологией и базовыми принципами. Arduino же просто сборище бибок, которые связывает страсть к `Begin` и `Loop`
  * имеет [шикарную документацию](https://docs.espressif.com/projects/esp-idf/en/v5.1.2/esp32s3/get-started/index.html), большая часть разделов которой сопровождается [полноценными примерами](https://github.com/espressif/esp-idf/tree/master/examples)

Конечно, у Arduino тоже есть свои плюсы, и было бы странно их отрицать:
  * кроссплатформенность и статус дефолтного фреймворка для многих MCU
  - отсюда несравнимо большее комьюнити
  - отсюда 1st class support разными производителями

Стоят ли эти преимущества сопутствующих ограничений и проблем, пусть каждый решает сам. Пробовал Arduino в [BoundBoxESP v0.0.1](https://github.com/buglloc/BoundBoxESP/tree/v0.0.1) и мне было не вкусно :'(

Но что-то я опять отвлекся, в общем виде текущая логика приложеньки выглядит так:
![bound-box-schematic](/assets/images/posts/hmfde/boundbox-esp-logic.excalidraw.png)Так мы:
  1. Инициализируем экран, сеть и прочую периферию
  2. Проверяем наличие прикопанных в [NVS](https://docs.espressif.com/projects/esp-idf/en/v5.1.2/esp32s3/api-reference/storage/nvs_flash.html) секретов (сейчас это SSH ключ и зигота мастер-ключа). В случае отсутствия - генерируем новые
  3. Дожидаемся подключения по сети (возможно зря, не решил пока)
  4. Запрашиваем PIN и генерируем мастер-ключ: `HMAC-SHA-256(NVSKey, EnteredPIN)`
  5. Затем тестовый секрет: `HMAC-SHA-256(MasterKey, VerificationSalt)`
  6. Спрашиваем ОК ли он
  7. Если нет, то возвращаемся к п4 и снова запрашиваем PIN
  8. Если все норм, то:
  - стартуем SSHD
  - обрабатываем входящие команды
  - показываем подписочные нотификашки при необходимости
  - ну вы поняли, работаем - запросы крутятся, секреты мутятся

Благодаря такой логике мы смогли получить три важных свойства:
  1. Прикованность мастер-ключа к конкретному устройству, т.к. один и тот же PIN на разных устройствах (точнее с разными NVSKey) породит разные мастер-ключи
  2. Резервируемость, благодаря хранению секретов в *зашифрованном* NVS и write-only работе с секретами. Так в случае беды мы можем:
  * налить новое устройство
  * загрузить в него старые секреты (если генерили ранее на хосте, офк)
  * спасти мир
  3. Антибрутальность, т.к. с т.з. стороннего наблюдателя любая пара PIN + NVSKey валидны. Да, тут чуть пострадал UX, зато не нужно хранить счетчик попыток ввода на флешке, дропать секреты при превышении и вот это все.

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

И для обычного работяги:
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

Ну и то, ради чего мы тут собрались - получение network-bound секрета:
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

Не стану затягивать и ныть о том, как и почему я отказался от использования [WolfSSH](https://github.com/wolfSSL/wolfssh) или о портировании [LilyGo-AMOLED](https://github.com/buglloc/BoundBoxESP/tree/2a3f212581254fd31d072d35ea22b18fabaf1f16/components/t_amoled) под ESP-IDF. Лучше вместо нытья демо покажу ^\_^, вот!

## В действии
Чтобы ничего не пропустить (и не забыть в будущем, ха-ха) - мы начнем с самого начала. Да-да, с клонирования репо и настройки под себя (i.e. логин/ключ админа):
  <div id="boundbox-init"></div>

На этом этапе мы должны получить полностью рабочую коробку, но я предлагаю еще чуть докинуть безопасности, усложнив получение данных из NVS. Для этого включаем релизное шифрование флешки (doc: [Flash Encryption: Release Mode](https://docs.espressif.com/projects/esp-idf/en/v5.1.2/esp32s3/security/flash-encryption.html#flash-enc-release-mode)) на "хостовом" ключе (doc: [Flash Encryption: Using Host Generated Key](https://docs.espressif.com/projects/esp-idf/en/v5.1.2/esp32s3/security/flash-encryption.html#using-host-generated-key)):
  <div id="boundbox-flash-encryption"></div>

На всякий случай - **использование хостового ключа обязательно**, т.к. я умышленно отказался от OTA, дабы быть ближе к принципу write-only секретов. Ну а пока мы получили шифрование дефолтных партиций и NVS. Осталось дело за [Secure Boot](https://docs.espressif.com/projects/esp-idf/en/v5.1.2/esp32s3/security/secure-boot-v2.html#how-to-enable-secure-boot-v2), т.к. иначе какой смысл в таком шифровании? `PARTITION_TABLE_OFFSET` я [потюнил заранее](https://github.com/buglloc/BoundBoxESP/blob/ec4ec6db7a2bca0f9969ff73336dcb58211c9f3e/sdkconfig.defaults#L12), поэтому места под [распухший бутлоадер](https://docs.espressif.com/projects/esp-idf/en/v5.1.2/esp32s3/security/secure-boot-v2.html#bootloader-size) нам должно без проблем хватить.

Погнали, нам нужно настроить бутлоадер, зашифровать его + partition table + приложеньку и прошить:
  <div id="boundbox-secure-boot"></div>

Вуаля! Теперь мы проверяем и подпись бутлоадера, и подпись приложеньки, что не позволит пограничнику загрузить собственные и считать все с флешки. Прямой же доступ мы ограничили ранее, за счет шифрования. Вот и чудненько, теперь главное ключи не потерять, хех :))

Не устали? Самое время полюбоваться итоговым результатом под присмотром Teo:
{% include video id="RFMkkGlY-y0" provider="youtube" %}

<script src="/assets/misc/asciinema-player.min.js"></script>
<script>
  AsciinemaPlayer.create(
    '/assets/videos/boundbox-init.cast',
    document.getElementById('boundbox-init'),
    {poster: 'npt:0:44'}
  );
  AsciinemaPlayer.create(
    '/assets/videos/boundbox-flash-encrypt.cast',
    document.getElementById('boundbox-flash-encryption'),
    {poster: 'npt:1:20'}
  );
  AsciinemaPlayer.create(
    '/assets/videos/boundbox-secure-boot.cast',
    document.getElementById('boundbox-secure-boot'),
    {poster: 'npt:0:42'}
  );
</script>

## Дальше-то что?
А дальше не торопливый выход в прод:
  - заиспользовать BoundBoxESP в не критичном сервисе
  - убрать валенки с пульта
  - собрать вторую копию BoundBoxESP (а может и v2)
  - ближе к майским перевести все нужные железки на FDE

Пожалуй, на этом пока все, всем кота (づ˶•༝•˶)づ♡