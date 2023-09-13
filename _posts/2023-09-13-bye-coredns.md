---
title: "ExternalDNS + AdGuard Home + CoreDNS => ExternalDNS + DNSGateway + AdGuard Home"
classes: wide
categories:
  - home-infra
---

До сегодня я использовал [CoreDNS](https://coredns.io/) как основной DNS сервер в домашней инфре. До сегодня, потому что его [Special Behaviour](https://coredns.io/plugins/etcd/#special-behaviour) в etcd плагине совсем не то, что ожидаешь увидеть по итогу. Я понимаю, почему они пошли по такому всратому пути, но от этого мне как-то не легче. Зацените сами: [CoreDNS+etcd vs DNS здорового человека](https://gist.github.com/buglloc/7f56e701d2adc36b8c02d9de9f5879e1)

### Да зачем вообще что-то?
Ну...смотрите сами:
  * у меня приличное количество (виртуальных и не очень) хостов дома, это какой-нить Jellyfin, qBittorrent, Unifi controller, HA, Proxmox, etc
  * всем им, как минимум, нужны:
    - понятные доменные имена
    - TLS сертификаты
  * хранятся эти все записи в [ExternalName]сервисах не менее домашнего k8s


Очевидно, что самое простое это завести какую-нить зону в Cloudflare (e.g. `buglloc.cc.`), заиспользовать его в качестве [провайдера](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/cloudflare.md) в [ExternalDNS](https://github.com/kubernetes-sigs/external-dns) и дело с концом. И это действительно так! Но только до тех пор пока ты не проводишь тракторные ученьки. Я их провожу, а значит без локального DNS не обойтись. В минимальном виде это просто резолвер с дисковым кешом, в моем случае это локальный авторитетник с кешом.

### Как было-то?
До сегодня это был ExternalDNS + [AdGuard Home](https://adguard.com/ru/adguard-home/overview.html) + CoreDNS. ExternalDNS в качестве control plane, AdGuard Home для красивых графичков + логов + фильтра, ну и CoreDNS как авторитетник с плюхами.
Вместо этого, можно было бы взять ченить в духе [Pi-hole](https://pi-hole.net/), но это жесть как скучно.
Вместо CoreDNS можно было бы взять, например, [PowerDNS](https://www.powerdns.com/) но я пока не познал его кунгфу. Или BIND, но я его принципиально не использую.

### А стало?
А с сегодняшнего дня это ExternalDNS + [DNSGateway](https://github.com/buglloc/DNSGateway) + AdGuard Home. В этой схеме DNSGateway выступает как оч простая прокся:
  - с одной стороны реализующая [RFC 2136](https://datatracker.ietf.org/doc/html/rfc2136) сервер с AXFR, в которую ExternalDNS пуляет обновления
  - с другой использующая [custom filtering rules](https://github.com/AdguardTeam/AdGuardHome/wiki/Hosts-Blocklists) из AdGuard Home в качестве бэкенда

Получившиеся локальные правила в AdGuard Home:
![Профит](/assets/images/posts/adguard-custom-rules.png)

BTW, DNSGateway я использую и для DNS-01 челленджей поверх Cloudflare , но это другая история