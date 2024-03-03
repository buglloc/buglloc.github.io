---
title: "ESPHome: T-Display S3 Long"
classes: wide
header:
  og_image: /assets/images/posts/t-display-s3-long-esphome/t-display-s3-long-esphome-cover.jpg
categories:
  - iot
  - esphome
---
Никогда ранее не писал компоненты для [ESPHome](https://esphome.io/), а тут судьба подкинула [T-Display S3 Long](https://www.lilygo.cc/products/t-display-s3-long), и пришлось. Именно об этом сегодняшняя зарисовочка ;)
<figure style="max-width: 800px">
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/posts/t-display-s3-long-esphome/t-display-s3-long-esphome-cover.jpg" alt="">
  <figcaption class="align-center"><a href="https://github.com/buglloc/esphome-components?tab=readme-ov-file#axs15231-display-wip" target="_blank">AXS15231 from <code>github://buglloc/esphome-components</code></a></figcaption>
</figure>
## T-Display S3 Long
Но для начала пара слов о [LILYGO T-Display S3 Long](https://www.lilygo.cc/products/t-display-s3-long):
<div>
<figure style="width: 400px" class="align-left">
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/posts/t-display-s3-long-esphome/s3-long-unboxing-0.jpg" alt="">
</figure>
<figure style="width: 400px" class="align-left">
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/posts/t-display-s3-long-esphome/s3-long-unboxing-1.jpg" alt="">
</figure>
<div class="cf"></div>
</div>

<div>
<figure style="width: 400px" class="align-left">
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/posts/t-display-s3-long-esphome/s3-long-unboxing-2.jpg" alt="">
</figure>
<figure style="width: 400px" class="align-left">
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/posts/t-display-s3-long-esphome/s3-long-unboxing-3.jpg" alt="">
</figure>
<div class="cf"></div>
</div>

На борту у которого:
  - ESP32-S3R8
  - 16MB флешки
  - 8MB PSRAM
  - зарядка для аккумулятора на базе [SY6970](https://github.com/buglloc/esphome-components/blob/main/docs/datasheet/SY6970.pdf)
  - 3.4" capacitive touch экран разрешением 180x640, QSPI интерфейсом и [AXS15231](https://github.com/buglloc/esphome-components/blob/main/docs/datasheet/AXS15231_Datasheet_V0.4_20221108.pdf) MCU

Это из хорошего, а из плохого:
  - ужасная (точнее практически отсутствующая) документация
  - кривые и полурабочие примеры

Не скажу, что у LILYGO документация когда-то была сильной стороной, но примерчики зачастую были норм. В этот же раз у меня сложилось впечатление, что никто и не ставил себе целью сделать что-то достойное, чисто запушили наброски какие были :( В общем, я бы скорее не советовал к покупке этот борд и проголосовал ~~батом~~рублем за что-то другое. Но что куплено, то куплено - тут главное не отчаиваться :)

## ESPHome: External Components
Как я и говорил в самом начале - использовать T-Display S3 Long я хотел из ESPHome. А ESPHome, будучи довольно взрослым фреймворком, поддерживает т.н. [External Components](https://esphome.io/components/external_components.html), с помощью которых можно как подключить новый компонент, так и заменить существующий.

 При этом процесс создания нового компонента не сложнее рисования совы:
  - заводим репо нужной структуры:

```bash
$ git init && mkdir -p components/<my_cool_component>
```
  
  - пишем новый компонент - см. [Contributing to ESPHome](https://esphome.io/guides/contributing#contributing-to-esphome)
  - все, любой желающий может подключить этот репо и воспользоваться компонентом:

```yaml
external_components:
  - source: github://cool-author/cool-repo-name
	components: [ my_cool_component ]
```

Если же говорить о потрошках самого компонента, то он состоит из трех частей:
  - [схема конфигурации](https://esphome.io/guides/contributing#config-validation) (нужна для валидации ямликов и какой-то типизации):

```py
import esphome.config_validation as cv

CONF_MY_REQUIRED_KEY = 'my_required_key'
CONF_MY_OPTIONAL_KEY = 'my_optional_key'

CONFIG_SCHEMA = cv.Schema({
  cv.Required(CONF_MY_REQUIRED_KEY): cv.string,
  cv.Optional(CONF_MY_OPTIONAL_KEY, default=10): cv.int_,
}).extend(cv.COMPONENT_SCHEMA)
```

  - [кодген из yaml в C++](https://esphome.io/guides/contributing#code-generation):

```py
import esphome.codegen as cg

def to_code(config):
    var = cg.new_Pvariable(config[CONF_ID])
    yield cg.register_component(var)

    cg.add(var.set_my_required_key(config[CONF_MY_REQUIRED_KEY]))
    if optional_val := config.get(CONF_MY_OPTIONAL_KEY):
        cg.add(var.set_my_optional_key(optional_val))
```

  - ну и сама реализация компонента на C++

Пожалуй, наиболее коварная часть - это одновременная поддержка нескольких нижележащих фреймворков, и это кунг-фу я, к своему стыду, пока не познал. При этом сам ESPHome с одной стороны имеет множество базовых абстракций (e.g. `I2CDevice`, `UARTDevice`, `SPIDevice`, etc), а с другой - внушительную базу примеров из уже реализованных компонентов.

## ESPHome: AXS15231 component
К счастью, один добрый человек уже добавил [поддержку QSPI в ESPHome](https://github.com/esphome/esphome/pull/5925), поэтому мне осталось только реализовать [Display](https://esphome.io/components/display/index.html) и [Touchscreen](https://esphome.io/components/touchscreen/index.html) компоненты.

Сказано - сделано. Пара утроночеров копипасты и [нужный компонент готов](https://github.com/buglloc/esphome-components/tree/main/components/axs15231). На 80% состоит из С++ и на 20% из Python, что считаю ожидаемым - ведь питоний там чисто в роли клея.

К сожалению, плохость примеров от LILYGO и моя тупость пока оставили за бортом такие фичи как:
  - DMA (скорее всего должен помочь ускорить рендеринг)
  - аппаратный поворот экрана (`MADCTL_MV` тупо все портит)
  - прерывания от сенсорного экрана

Не скажу, что эти фичи мне прям сейчас очень нужны, но как-то обидненько, и если будут силы, то я добью. Но хватит болтовни! Подключаем борд к ESPHome, пишем простенький конфиг (внимательнее с пинами, как обычно):
```yaml
esphome:
  name: t-display-s3-long
  friendly_name: LilyGo-T-Display-Long
  platformio_options:
    upload_speed: 921600
    build_unflags: -Werror=all
    board_build.flash_mode: dio
    board_build.f_flash: 80000000L
    board_build.f_cpu: 240000000L

psram:
 mode: octal
 speed: 120MHz

esp32:
  board: esp32-s3-devkitc-1
  flash_size: 16MB
  framework:
    type: esp-idf

external_components:
  - source: github://buglloc/esphome-components
    refresh: 10min

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

spi:
  id: display_qspi
  clk_pin: 17
  data_pins:
    - 13
    - 18
    - 21
    - 14

i2c:
  sda: 15
  scl: 10
  id: touchscreen_bus

font:
  - file: "gfonts://Madimi One"
    id: font_std
    size: 40
    glyphs: "!\"%()+=,-_.:°0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ abcdefghijklmnopqrstuvwxyz/\\[]|&@#'"
  - file: "gfonts://Madimi One@400"
    id: font_title
    size: 40
    glyphs: "!\"%()+=,-_.:°0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ abcdefghijklmnopqrstuvwxyz/\\[]|&@#'"

globals:
  - id: title_color
    type: Color
    initial_value: "Color::random_color()"

display:
  - platform: axs15231
    id: main_display
    spi_id: display_qspi
    dimensions:
      height: 640
      width: 180
    cs_pin: 12
    reset_pin: 16
    backlight_pin: 1
    transform:
      swap_xy: false
    rotation: 90
    auto_clear_enabled: false
    lambda: |-
      it.print(it.get_width()/2, it.get_height()/2-20, id(font_title), id(title_color), TextAlign::CENTER, "ESPHome @ AXS15231B");
      it.print(it.get_width()/2, it.get_height()/2+30, id(font_std), {230,0,115}, TextAlign::CENTER, "@UTBDK");

touchscreen:
  - platform: axs15231
    id: main_touch
    display: main_display
    i2c_id: touchscreen_bus
    transform:
      swap_xy: true
    on_touch:
      - lambda: |-
            Color newColor;
            do { newColor =  Color::random_color(); } while (newColor == id(title_color));
            id(title_color) = newColor;

            ESP_LOGI("cal", "x=%d, y=%d, x_raw=%d, y_raw=%0d",
              touch.x,
              touch.y,
              touch.x_raw,
              touch.y_raw
            );
```

Прошиваем, запускаем и....вуаля:
{% include video id="OjGWA1THGsI" provider="youtube" %}

## Closure
Признаться, у меня пока настолько положительные впечатления от ESPHome, что в следующий раз придется хорошенько задуматься - тащить в очередной проект поддержку mqtt, ota, ha, wifi manager и черта лысого руками или дописать недостающее внешним компонентом к ESPHome. Понятное дело, что проект проекту рознь, но для большей части сенсоров, датчиков, кнопочек и прочего home automation IoT я бы точно рассматривал ESPHome как добротную платформу :)

А на этом у меня все, всем кота (づ˶•༝•˶)づ♡
