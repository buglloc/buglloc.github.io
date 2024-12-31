---
title: "KVMBattle: KVMZer0 vs KVMR0ck vs NanoKVM-W"
classes: wide
header:
  og_image: /assets/images/posts/kvmbattle/kvmbattle-cover.png
categories:
  - diy
  - home-infra
---
Кратенький обзор получившихся KVM. Поговорим о том, зачем, почем и вообще :)
![cover](/assets/images/posts/kvmbattle/kvmbattle-cover.png){: .align-center}

## Зачем?
Это, к счастью, самый простой ответ - затем. Затем, что у меня разбросано всякое оборудование по дому, и тащить монитор с клавиатурой к какому-нить MiniPC, стоящему в шкафу не оч приятное занятие. Плюс, KVM в целом сильно удобнее клавиатуры с монитором, потому что работает вставка текста, скрипты, есть виртуальный диск и т.д. Ну знаете как бывает - сеть настроить, пароль из KeePassXC вставить, на kernel panic посмотреть и тд.

И если со всевозможными SBC зачастую сильно легче, ибо там чаще всего торчит UART, и лучше обойтись каким-нить бриджом на ESP32 (у меня для этого есть крошка [M5Stack NanoC6](https://shop.m5stack.com/products/m5stack-nanoc6-dev-kit) - как-нить напишу и о нем, наверное). То со всякими MiniPC все сильно сложнее.

Поэтому мне и захотелось иметь какой-то KVM и не просто KVM, а с рядом требований:
- до $100
* не требующий ядерного реактора для работы
* с поддержкой WiFi

Ограничение по цене сразу отметает все дорогие (среди дешевых) варианты с RPi4 на борту в духе [PiKVM V3](https://a.aliexpress.com/_oDD5by7) или [TinyPilot Voyager 2a](https://tinypilotkvm.com/product/tinypilot-voyager2a). А ограничение по питанию и подавно. Благо есть множество способов уложиться в $100 - будем выкручиваться ;)
Потому пора представить вам три варианта, которые я собрал. Встречайте...

## KVMZer0
[KVMZer0](/kvmzer0/) - самый дешевый KVM, когда из "говна и палок" остались только палки:
![cover](/assets/images/posts/kvmbattle/kvmzer0.jpg){: .align-center}

Собран из [Orange Pi Zero 2W](http://www.orangepi.org/html/hardWare/computerAndMicrocontrollers/details/Orange-Pi-Zero-2W.html) c [HDMI to USB](https://a.aliexpress.com/_opPvKMx). По [ссылке](/kvmzer0/) найдете детальное описание, а сейчас нам лишь важно, что на все про все ушло в районе $20.

## KVMR0ck
[KVMR0ck](/kvmr0ck/) уже сильно недешевый KVM:
![cover](/assets/images/posts/kvmbattle/kvmr0ck.jpg){: .align-center}

С ним я не очень экономил, поэтому выложил около $70 за [Radxa ZERO 3W](http://radxa.com/products/zeros/zero3w/) + [HDMI-CSI](https://a.aliexpress.com/_olvAXLh). По [ссылке](/kvmr0ck/), как всегда, найдете детальное описание.

## NanoKVM-W
[NanoKVM-W](/nanokvm-w/) - WiFi мод для [NanoKVM](https://sipeed.com/nanokvm) от Sipeed:
![cover](/assets/images/posts/kvmbattle/nanokvm.jpg){: .align-center}

Стоил примерно $50-$60 (смотря как считать) и это яркий представитель:
![cover](/assets/images/posts/kvmbattle/devices.jpg){: .align-center}

По [ссылке](/nanokvm-w/) вас ждет детальное описание, процесс сборки и настройки.

## Fight!
В качестве тестового стенда у нас будет какой-то ноутбук с виндой и power bank как источник питания для KVM:
![cover](/assets/images/posts/kvmbattle/stand.jpg){: .align-center}

А в качестве софта [PiKVM](https://pikvm.org/) у `KVMZer0`/`KVMR0ck` и [NanoKVM](https://github.com/sipeed/NanoKVM) у `NanoKVM`. PiKVM можно было бы заменить на [TinyPilot](https://github.com/tiny-pilot/tinypilot), но я не нашел для себя преимуществ последнего (кроме желания денег за поддержку HTTPS офк).

Забегая вперед, скажу, что [KVMZer0](/kvmzer0/) с этим испытанием не справился - не хватило питания и пришлось подключать его к заряднику. Ну да как говорила моя бабушка - лучше один раз увидеть, а бабушку нужно слушать:
{% include video id="1043068228" provider="vimeo" %}

## Closure
Напомню, что KVM мне по большей части нужен для всякого рода траблшутинга, потому особых требований к отзывчивости я не имею. С пингом 200ms в сторону работы быстро привыкаешь сначала думать - потом писать ;)

И с этой позиции я считаю так:
  - если вам не очевидна полезность KVM и вы готовы питаться от сети - соберите что-то дешевое с USB HDMI. Попробуйте, поиграйте, а дальше решите. В крайнем случае переделаете в ambient light для приставки ;)
  - если вам критичен input lag и хочется всяких ништяков - собирайте с HDMI-CSI bridge. Будет недешево, но это ваш выбор. Бонусом получите мощную железку в кармане
  - если хочется новых впечатлений - берите NanoKVM. Этому проекту все еще не хватает любви и ласки, но я верю в его успех

Но не верьте одному мнению - посмотрите/почитайте другие посты на тему. Для себя же я пока выбрал NanoKVM, т.к. он чертовски мил и технически интересен. Но это мой выбор, я пока не могу его советовать всем и каждому.

А пока у меня все, всем кота (づ˶•༝•˶)づ♡
