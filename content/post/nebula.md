---
title: Nebula
date: "2022-10-21T07:36:06Z"
author: bubnovd
authorTwitter: bubnovdnet
image: "/img/nebula/logo.jpg"
description: Nebula - overlay сеть от разработчиков Slack в качестве замены традиционных VPN сетей
tags:
- VPN
- overlay
- Nebula
keywords:
- VPN
- nebula
- network
- overlay
showFullContent: false
readingTime: true
hideComments: false
---

_Обложка: Млечный Путь с хижины Рацека 24.09.2022_

В наше время о VPN знает каждый школьник и пенсионер. Но не все знают, что эта технология зародилась вовсе не для обхода блокировок. 

__TL;DR__
В качестве VPN можно использовать overlay решения. Они ещё недостаточно изучены борцами за "чистоту" Интернета и часто настраиваются намного проще традиционных VPN решений. А ещё overlay позволяет объединять хосты из разных облачных провайдеров, настраивать роутинг и фаервол прямо из конфига

# VPN 
_Сейчас будет очень упрощенно. Сетевиков прошу пропустить этот пункт_

VPN - Virtual Private Network - общее название технологий, объединяющих немаршрутизируемые сети с использованием других сетей. Долгое время основным назначением VPN было объединение офисов, расположенных в разных городах/странах/континентах. Как правило, организации имеют локальную сеть, связанную с Интернетом через роутер посредством NAT'a. Это значит, что любой хост из локалки выходит в Интернет всегда с одним адресом - внешним адресом роутера. Проблема в том, что адреса внутри локальной сети немаршрутизируемы в большом Интернете. То есть нельзя просто взять и пингануть с 192.168.1.11 узел 192.168.2.12. Такой пакет просто дропнется первым же роутером в Интернете

![vpn1](/img/nebula/vpn1w.png)

Эта задача решается с помощью виртуальных сетей: поверх сущществующих байтов, летящих от хоста А к хосту Б просто доавляются дополнительные байты - заголовки, с помощью которых можно найти путь между сетями, которые не могут быть доступны в обычном мире

![vpn2](/img/nebula/vpn2w.png)

Если совсем грубо - то к пакету добавляются дополнительные заголовки, позволяющие ему долететь до нужно точки в Интернете, отбросить эти заголовки и уже попасть к нужному хосту

![vpn3](/img/nebula/vpn3w.png)

В современном мире применение VPN расширилось, хотя технологии и принципы остались прежними. Для обычного человека вместо удаленного офиса просто появился Интернет, в который нужно попасть через удаленный узел, у которого совсем другой IP

# Overlay

В ИТ всё построено вокруг абстракций и иллюзий: 
- модель OSI, где разные уровни сети вложены друг в друга и каждый обеспечивает свои функции
- страницы виртуальной памяти, эффективно использующие память физическую
- прерывания, незаметно передающие ресурсы процессора между процессами

Overlay и Underlay - всего лишь ещё один способ разделить разные уровни сети по их функциональности. 
- Underlay - "традиционная" сеть, состоящая из физических кабелей, коммутаторов, маршрутизаторов и логических протоколов, пакетов, правил маршрутизации
- Overlay - "надсеть". Дополнительный уровень сети, работающий поверх Underlay. Поверх TCP можно запустить протокол, соединяющий хосты и настраивающий свои способы управления трафиком: маршрутизацию, приоритезацию, фильтрацию трафика. VPN тоже можно назвать оверлеем, так как он создает новую сеть поверх существующей. Типичные overlay технологии: VXLAN, VPLS

Изначально термин overlay применялся для больших сетей, таких как сети ЦОД. Для объединения двух ЦОДов уже недостаточно просто запустить пару туннелей между ними. Наступила эра контейнеров. Сервис может переезжать с одной машины на другую, меняя не только свой адрес, но и целые континенты. Сети состоят из сервисов, развернутых у разных облачных провайдеров, тянуть VPN между которыми может быть невозможно. Тут и приходят на помощь overlay сети: overlay агент может быть развернут на каждой машине и соединяться только с нужными хостами, грамотно управлять трафиком, подстраиваться под изменения

![overlay](/img/nebula/overlay.svg)

# Nebula

Nebula - overlay сеть, разработанная компанией Slack - той самой, которая сделала знаменитый мессенджер. Slack выложил Nebula в Open Source. После чего ключевые разработчики Nebula - Ryan Huber и Nate Brown - основали [свою компанию](https://www.defined.net) и теперь развивают Nebula и занимаются внедрением и поддержкой. В их блоге можно найти интересные материалы про overlay

{{< youtube qy2cgqglt3o >}}


Nebula агент работает на каждом хосте в сети и создает mesh сеть. Mesh это такая топология, где каждый хост соединен с каждым. То есть в сети нет единой точки обмена трафиком. Чтобы отправить пакет от хоста A к хосту B не нужен промежуточный узел - каждый узел имеет прямой путь к любому другому. Так же хост A может отправить пакет на любой другой хост в меш сети.

![mesh](/img/nebula/meshw.png)

В Nebula узлы делятся на два типа:
- управляющий. В терминах Nebula он называется Lighthouse. Этот тип узла содержит инфрмацию обо всей сети. Каждая нода подключается к lighthouse и узнает от него о других нодах. В сети может быть несколько lighthouse нод. Управляющая нода потребляет минимум ресурсов, единственный важный нюанс - между нодами и lighthouse должен маршрутизироваться трафик (читай - нужен белый IP для lighthouse)
- рядовая нода. Нода, отправляющая или принимающая трафик. После подключения к lighthouse она знает обо всех нодах в сети и способна отправить трафик напрямую на любую из них

Ещё один интересный момент - работа за натом. Если есть два хоста, находящихся за разными натами в разных локальных сетях, то Lighthouse может соединить их друг с другом напрямую. Опять же, трафик будет ходить напрямую - минуя третьи узлы. Эта технология называется [UDP Hole Punching](https://en.wikipedia.org/wiki/UDP_hole_punching). Если же хосты окажутся в одной локалке, логично, что трафик между ними тоже не будет вылетать за пределы локальной сети (mesh же, вы помните?)

![lighthouse](/img/nebula/lighthousew.png)

# Configuration

Рассмотрим настройку Nebula в качестве замены обычным VPN решениям для ~~обхода блокировок~~ доступа в удаленные сети. В качестве выходной ноды будем использовать Lighthouse. Подключаться к нему будем с Ubuntu 22.04 и Android, А МОЖЕТ ЕЩЁ И С ВНИДЫ??

Для Fedora и Arch есть готовые пакеты, для остальных скачиваем архив с [GitHub](https://github.com/slackhq/nebula/releases) и распаковываем его
Вся Nebula это два бинарника:
- nebula-cert - управляет сертификатами для подключения
- nebula - собственно сам агент
Кладем их в /usr/local/bin

О топологии сети надо подумать до генерации сертификатов, так как в сертификате указываются группы хостов, их адреса и подсети, к которым разрешен доступ владельцу сертификата. В нашем случае будет хоста: ДВА?? А КАК ЖЕ АНДРОЕД
- lighthouse. Он же выходная нода. Через него будет уходить трафик в интернет. Ему нужен белый адрес. Я арендовал машинку на самом дешевом хостинге. Внешний адрес `198.18.123.11`, внутренний в overlay `192.168.255.1`
- laptop. Хост, который будет ходить в интернет через lighthouse. Overlay адрес `192.168.255.2`

> В нормальных условиях для соединения хостов между собой не нужно пускать трафик через lighthouse. Вообще, lighthouse должен заниматься только управлением overlay сетью - не надо заворачивать на него трафик. Мы это делаем только потому что наша задача максимальна простая и не требует никакой безопасности - тупо пустить трафик с одного хоста через другой



## Выходная нода
В моём случае lighthouse запущен на хосте с Ubuntu 20.04. Он же является нодой для выхода трафика во внешний мир. В идеале трафик через lighthouse ходить не должен. Но я принимаю эти риски

- Включаем ip_forwarding `sudo cat "net.ipv4.ip_forward = 1" > /etc/sysctl.conf`. Перечитываем изменнные настройки `sudo sysctl -p`
- Включаем MASQUERADING  `sudo iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE`, где `ens3` - интерфейс, смотрящий в интернет
- Не забудьте сохранить конфигурацию iptables: [iptables-save, iptables-restore](https://linuxhint.com/iptables-rules-linux-iptables-save-command/)

## Генерируем сертификаты

- CA: `./nebula-cert ca -name "Myorganization, Inc"`
- Lighthouse: `./nebula-cert sign -name "lighthouse" -ip "192.168.255.1/24" -subnets "0.0.0.0/0"` subnets тут - подсети, трафик на которые будет ходить через lighthouse
- Laptop: `./nebula-cert sign -name "laptop" -ip "192.168.255.2/24" -groups "laptop,home"`

## Lighthouse

Вся конфигурация Nebula лежит в одном конфиг файле. Он будет немного различаться на разных нодах
Создаем директорию `/etc/nebula`, кладем туда полученные ранее `ca.crt`, `lightouse.crt`, `lighthose.key`
И `config.yaml`:
```
pki:
  ca: /etc/nebula/ca.crt
  cert: /etc/nebula/lighthouse.crt
  key: /etc/nebula/lighthouse.key
lighthouse:
  am_lighthouse: true
  interval: 60
listen:
  host: 0.0.0.0
  port: 4242
punchy:
  punch: true
relay:
  am_relay: false
  use_relays: true
tun:
  disabled: false
  dev: nebula1
  drop_local_broadcast: false
  drop_multicast: false
  tx_queue: 500
  mtu: 1300
  routes:
  unsafe_routes:
logging:
  level: info
  format: text
firewall:
  conntrack:
    tcp_timeout: 12m
    udp_timeout: 3m
    default_timeout: 10m
  outbound:
    - port: any
      proto: any
      host: any
  inbound:
    - port: any
      proto: any
      host: any
```

Детальное описание конфига доступно в [GitHub](https://github.com/slackhq/nebula/blob/master/examples/config.yml)

Чтобы оверлей запускался всегда, создаем systemctl unit:
```
sudo cat > /usr/local/lib/systemd/system/nebula.service

[Unit]
Description=nebula
Wants=basic.target
After=basic.target network.target

[Service]
SyslogIdentifier=nebula
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/usr/local/bin/nebula -config /etc/nebula/config.yaml
Restart=always

[Install]
WantedBy=multi-user.target
```

`sudo systemctl daemon-reload && sudo systemctl enable --now nebula`

Проверяем, что все запустилось
```
systemctl status nebula
● nebula.service - nebula
     Loaded: loaded (/usr/local/lib/systemd/system/nebula.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2022-10-18 16:48:15 UTC; 2 days ago
   Main PID: 25252 (nebula)
      Tasks: 11 (limit: 1102)
     Memory: 15.1M
     CGroup: /system.slice/nebula.service
             └─25252 /usr/local/bin/nebula -config /etc/nebula/config.yaml

```

## Laptop

Настройка аналогична lighthouse:
- кладем бинарники в /usr/local/bin
- сертфикаты `ca.crt`, `laptop.crt`, `laptop.key` в `/etc/nebula`
- создаем и запускаем systemd unit. Сейчас он не запустится, потому что конфига нет

`config.yaml`:
```
pki:
  ca: /etc/nebula/ca.crt
  cert: /etc/nebula/ibs.crt
  key: /etc/nebula/ibs.key
static_host_map:
  "192.168.255.1": ["198.18.123.11:4242"]
lighthouse:
  am_lighthouse: false
  interval: 60
  hosts:
    - "192.168.255.1"
listen:
  host: 0.0.0.0
  port: 4242
punchy:
  punch: true
relay:
  am_relay: false
  use_relays: false
tun:
  disabled: false
  dev: nebula1
  drop_local_broadcast: true
  drop_multicast: true
  tx_queue: 500
  mtu: 1300
  routes:
  unsafe_routes:
    - route: 0.0.0.0/0
      via: 192.168.255.1
logging:
  level: info
  format: text
firewall:
  conntrack:
    tcp_timeout: 12m
    udp_timeout: 3m
    default_timeout: 10m
  outbound:
    - port: any
      proto: any
      host: any
  inbound:
    - port: any
      proto: icmp
      host: any
```

`sudo systemctl daemon-reload && sudo systemctl enable --now nebula && systemctl status nebula`

Проверяем, что трафик ходит через Overlay:
```
mtr -n 8.8.8.8

Host:
1. 198.18.123.1
...
```

В трассировке первым хопом должен быть адрес шлюза lighthouse. Если это так - поздравляю, overlay настроен!

В такой конфигурации весь трафик будет заворачиваться в Overlay. Включая трафик до lighthouse. Поэтому туннель упадет сразу после установки. Чтобы избежать этого нужно добавить маршрут до lighthouse через провайдера
`sudo ip route add 198.18.123.11/32 via 192.168.0.1`

---
Тут описано не совсем обычное применение Nebula, поэтому возможны спецэффекты при таком способе применения. Всё-таки эта технология предназначена для соединения огромного количества узлов, расположенных в разных местах в единую сеть с прямым доступом между узлами, хорошей масштабируемостью и простой настройкой. Я выбрал Nebula для VPN просто чтобы познакомиться с чем-то новым

---
Ещё статьи по теме по тегам [MTU](/tags/mtu)  [VPN](/tags/vpn)