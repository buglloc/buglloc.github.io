---
title: "DNSGateway + Cloudflare to issue Let's Encrypt certificates with DNS-01 challenges"
classes: wide
categories:
  - home-infra
---

До сегодня я переживал за свои [Cloudflare](https://www.cloudflare.com/) токены разбросанные по кучке машин. Благо не так давно я начал писать [DNSGateway](https://github.com/buglloc/DNSGateway), позволяющий [более]гранулярно управлять правами на DNS зону, а сегодня закончил пееезд на него :)

![dnsgateway](/assets/images/posts/dnsgateway-overview.svg)

### Intro
Как и многие в последние годы я стараюсь всегда настраивать https для сервисов - будь-то домашний или публичный вебчик, будь-то консоль роутера или какой-нить ESPHome. Во-первых, потому что к этому всячески принуждают браузеры (ограничение фичей, страшные/бесячие предупреждения и т.д.). Во-вторых, потому что это стоит примерно ничего и по деньгам, и по времени. Примерно ничего, кроме проблем с безопасностью, все как мы любим :)

Но, наверное, для начала нужно дать немного ~~заурядных~~ классических вводных:
  - NSами я живу в [Cloudflare](https://www.cloudflare.com/). Мне нравится их относительная независимость + скорость + поддерживаемое API
  - Сертификаты мне нужны на пачке роутеров, нескольких ingress виртуалках, cert-manager'е k8s и бог знает где еще
  - В качестве Certificate Authority, конечно же, использую [Let’s Encrypt](https://letsencrypt.org/). Была мысль развернуть собственный, но здравый смысл победил эту идею

А коль у меня в качестве CA используется Let’s Encrypt, то и ACME DNS-01 челленджи идеально вписываются в общую картинку:
  - поддерживаются вайлкарды
  - не нужно мутить туннели/DNS View/прочие костыли для HTTP доступа со стороны Let’s Encrypt
  - на текущий момент поддержаны если не везде, то почти везде

Не стану перепечатывать тут RFC, стоит разок прочесть в оригинале (хотя б кусочек ^_^):
  - [ACME: DNS Challenge (RFC 8555)](https://datatracker.ietf.org/doc/html/rfc8555#page-66)
  - [Let’s Encrypt: DNS-01 challenge](https://letsencrypt.org/docs/challenge-types/#dns-01-challenge)

Но в общем случае DNS-01 challenge выглядит так:
  1. клиент хочет пруфнуть владеление доменом `lol.cheburek` для запроса сертификата
  2. ACME сервер выдает список челленджей
  3. клиент добавляет TXT запись `_acme-challenge.lol.cheburek` и сообщает серверу "я сделяль"
  4. ACME сервер убеждается в корректности TXT записи и позволяет выписать сертификат

И тут становится очевидной проблема, ради которой я [DNSGateway](https://github.com/buglloc/DNSGateway) и затеял - при запросе сертификата должен быть доступ в Cloudflare DNS для добавления/удаления TXT записи `_acme-challenge.<domain>`, что и пруфнет владение доменом. К сожалению, в бесплатном тарифе (на самом деле я хз, читал на каком-то форуме, что в платных это не так) Cloudflare умеет ограничивать API токены только до зоны целиком. Т.е., например, вы не можете создать API токен, который может управлять только TXT записями или отдельными поддоменами. А это означает, что токен с полноценным доступом в DNS зону лежит на рандомных тачках, что не внушает оптимизма.

### Варианты решения
Фуф, когда проблематика ясна, можно перейти к вариантам решения. Оговорюсь, что это навряд ли полный перечень, скорее те варианты, которые я рассматривал для себя.

##### Варик для крохотной инфры: заиспользовать Throwaway Domain (по сути CNAME в треш-домен) + [ACME-DNS](https://github.com/joohoi/acme-dns)
В целом, идея [acme-dns](https://github.com/joohoi/acme-dns) мне оч понравилась. И, наверное, будь у меня один wildcard, я бы на этом и остановился. Но попробовав перетащить несколько машинок на acme-dns, я осознал, что подход с [DNS Alias](https://github.com/acmesh-official/acme.sh/wiki/DNS-alias-mode)/[Throwaway Domain](https://www.eff.org/deeplinks/2018/02/technical-deep-dive-securing-automation-acme-dns-challenge-validation) требует сильно больше сил, чем я рассчитывал :/

##### Варик для смекалистых: auth proxy
Я представлял себе его так:
  - мимикрируем под Cloudflare API
  - парсим запрос + проверяем аутентификацию/авторизацию
  - если все норм, то докинули API Token и отправили запрос в апстрим

Из очевидных плюсов - не нужно ничего патчить на клиенте.

Из минусов, которые погубили для меня эту идею:
  - слишком багоопасно
  - не всякий клиент позволяет конфигурить альтернативный апстрим

##### Варик не для ленивых: все сам, все сам
Чтож, коль acme-dns мне не подошел, а auth proxy больно попахивает, остается только засучить рукава и:
  - написать кастомный сервис поверх Cloudflare
  - к нему плагин для [certbot](https://certbot.eff.org/)/[acme.sh](https://github.com/acmesh-official/acme.sh)/etc
  - профит

Так-то оно так, но писать плагины под все возможные клиенты я очень не хотел. Поэтому было решено двигаться дальше и искать возможность реализации какого-нить wellknown сервера.

##### Мой выбор: DNS Update (RFC2136)
И варик с wellknown сервером нашелся, им стал [DNS Update (RFC2136)](https://datatracker.ietf.org/doc/html/rfc2136):
  - у него хорошая поддержка со стороны клиентов, например:
    * [acme.sh: Use nsupdate to automatically issue cert](https://github.com/acmesh-official/acme.sh/wiki/dnsapi#7-use-nsupdate-to-automatically-issue-cert)
    * [certbot: dns_rfc2136 plugin](https://certbot-dns-rfc2136.readthedocs.io/en/stable/)
    * [cert-manager: RFC-2136](https://cert-manager.io/docs/configuration/acme/dns01/rfc2136/)
  - будучи не молодым RFC, у него есть реализации серверной стороны. Например, от [miekg@](https://github.com/miekg/dns)

Таким образом, мы собираем одни плюсы:
  - [x] править acme клиенты не нужно
  - [x] имеет из коробки аутентификацию/авторизацию поверх TSIG
  - [x] не приходится отказываться от Cloudflare, плодить зоны или править оную руками
  - [x] токен от Cloudflare хранится в одном месте

О реализции ниже, не теряйтесь :)

### DNSGateway: описание
Сказано - сделано! Так я и написал для себя [DNSGateway](https://github.com/buglloc/DNSGateway). Он же, кстати, обеспечивает мне и синк [локальной зоны в AdGuard Home](https://ut.buglloc.com/home-infra/bye-coredns/), что тоже повлияло на принятие решения о необходимости заиметь его ;)

Технически DNSGateway представляет из себя [DNS Update (RFC2136)](https://datatracker.ietf.org/doc/html/rfc2136) сервер, который:
  - поддерживает TSIG аутентификацию/авторизацию с по-клиентным:
    * разграничением доступа к той или иной зоне. Например, разрешает управлять только `foo.buglloc.cc` и `bar.buglloc.cc`, но не `baz.buglloc.cc`
    * ограничением по типам записей. Например, позволяет создавать только TXT записи
    * опциональной поддержкой XFR (для ACME не нужно, но полезно)
  - умеет использовать в качестве бэкенда Cloudflare или AdGuard Home

Процесс получения сертификата с DNSGateway не стал слишком сложнее. Для наглядности:
![dnsgateway](/assets/images/posts/dnsgateway-overview.svg)

Если точнее, то когда один из acme-клиентов (роутер, cert-manager, etc) хочет получить сертификат он:
  - получает DNS-01 челлендж
  - добаляет его в DNSGateway, который:
      * проверит аутентификацию/авторизацию для пары TSIG key + zone
      * проверит можно ли клиенту создавать TXT записи
      * сходит в Cloudflare, дабы добавить запись
   * получает сертификат и удаляет TXT запись с помощью DNSGateway, который в свою очередь также сходит в Cloudflare

Звучит не плохо, ммм?

### DNSGateway: примерчики

В качестве минимального примера будет получение сертификата с помощью certbot на bookworm.
Для начала сконфигурим DNSGateway:
```yaml
listener:
  kind: rfc2136
  rfc2136:
    addr: :5454
    clients:
      - name: tst.
        secret: NzBjOTU4OTVlOTZlOTg5OGQwYTUxYTdjNWYzNTI3NzA5YjIyZTIxNWVjOTc3NWMxNzIxZjdjN2ExNjliNDc1ZCAgLQo=
        xfr_allowed: false
        zones:
          - example.buglloc.cc.
        types:
          - txt

upstream:
  kind: cloudflare
  cloudflare:
    zone_id: c74b3a3b1002eba34d6cb7f8a62b69da
    token: 3f786850e387550fdab836ed7e6dc881de23001b
```

Тут мы видим, что:
  - у нас один клиент `tst.`
  - ему разрешено править TXT записи в зоне `example.buglloc.cc.`
  - в качестве апстрима, конечно же, используется Cloudflare

Теперь ставим все нужные пакетики на виртуалочку:
```bash
$ sudo apt install -y certbot python3-certbot-nginx python3-certbot-dns-rfc2136
```

Создаем конфиг для dns_rfc2136 плагина:
```bash
$ sudo mkdir -p /etc/certbot
$ sudo touch /etc/certbot/rfc2136.ini && sudo chmod 600 /etc/certbot/rfc2136.ini
$ sudo tee -a /etc/certbot/rfc2136.ini <<EOT
# Target DNS server (IPv4 or IPv6 address, not a hostname)
dns_rfc2136_server = 10.10.10.3
# Target DNS port
dns_rfc2136_port = 5454
# TSIG key name
dns_rfc2136_name = tst.
# TSIG key secret
dns_rfc2136_secret = NzBjOTU4OTVlOTZlOTg5OGQwYTUxYTdjNWYzNTI3NzA5YjIyZTIxNWVjOTc3NWMxNzIxZjdjN2ExNjliNDc1ZCAgLQo=
# TSIG key algorithm
dns_rfc2136_algorithm = HMAC-SHA256
# TSIG sign SOA query (optional, default: false)
dns_rfc2136_sign_query = true
EOT
```

И заказываем сертификат:
```bash
$ sudo certbot certonly --dns-rfc2136 --dns-rfc2136-credentials /etc/certbot/rfc2136.ini --dns-rfc2136-propagation-seconds 30 -d example.buglloc.cc -d '*.example.buglloc.cc'
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Requesting a certificate for example.buglloc.cc and *.example.buglloc.cc
Waiting 30 seconds for DNS changes to propagate

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/example.buglloc.cc/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/example.buglloc.cc/privkey.pem
This certificate expires on 2023-12-16.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.
We were unable to subscribe you the EFF mailing list because your e-mail address appears to be invalid. You can try again later by visiting https://act.eff.org.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

Для визуалов:
![certbot example](/assets/images/posts/dnsgateway-certbot-example.png)

На этом как бы и все, всем TLS :)

### Closure

На всякий случах, хотел бы обратить внимание, что я бы пока не советовал завязываться на DNSGateway, потому что он находится на слишком раней стадии разработки. А если и завязываться, то быть готовому к поломкам и валенкам :)

BTW, таким же образом можно и полноценно распилить управление зоной. Мне просто это не нужно, т.к. она управляется из нодго единого места. Но потенциал имеется ;)