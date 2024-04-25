+++
title = "Блог переехал"
date = "2022-09-02T04:02:52Z"
author = "bubnovd"
authorTwitter = "bubnovdnet"
cover = ""
tags = ["history", ""]
keywords = ["", ""]
description = "История переезда и планы на новый блог"
showFullContent = false
readingTime = true
hideComments = false
+++

Блог переехал на платформу GitHub с гугловского Blogger. Кажется, что Blogger безнадежно устарел или я просто поленился с ним разбираться.
Сейчас сайт работает на [Hugo](https://gohugo.io/) с темой [cleanwhite](https://github.com/zhaohuabing/hugo-theme-cleanwhite.git).

Все статьи старого блога переехали сюда в секцию [blogger](/blogger). Самые популярные из них остались на прежних адресах, то есть ссылки на них и закладки продолжат работать.



## История

- Этот блог начинался как записная книжка сисадмина
- Первое сообщение было опубликовано 26 сентября 2010
- Посты были обо всем, касающемся ИТ: Windows, Linux, сети, безопасность, ссылки на интересности, найденные в сети
- С 2013 неожиданно повалил трафик. Причиной этому _внезапно_ стал [pikabu](https://pikabu.ru/story/salyut_burzhua_1577076?cid=16396660)
- В 2015 я апгрейднулся до сетевика и стал плотно работать с Mikrotik. Большинство постов стало о сетях и RouterOS
- В среднем в месяц на блог заходили 15 тысяч человек (или роботов. Я не пытался разбираться откуда трафик)

### Популярные посты
- [IPSec over L2TP между RouterOS и Apple iOS 10 ](http://www.bubnovd.net/2016/10/ipsec-over-l2tp-routeros-apple-ios-10.html) - почти 40000 просмотров
- [Почему за EoIP over OpenVPN нужно отрубать руки. И почему обе.](http://www.bubnovd.net/2016/01/eoip-over-openvpn.html) - 32400 просмотров
- Пост [Маршрутизация. OSPF в Mikrotik RouterOS](http://www.bubnovd.net/2016/03/ospf-mikrotik-routeros.html) просмотрело почти 29 тысяч человек
- [Система управления конфигурациями Oxidized ](http://www.bubnovd.net/2017/10/oxidized.html) - 23000 просмотров
- [DHCP Failover with RouterOS ](http://www.bubnovd.net/2017/07/dhcp-failover-with-routeros.html) - 9000 просмотров
- [Сегментация сети с использованием оборудования Mikrotik и D-Link ](http://www.bubnovd.net/2015/12/mikrotik-d-link.html) 6000 просмотров
- [Самый полный мануал по резервированию интернета на Mikrotik RouterOS](http://www.bubnovd.net/2015/03/mikrotik-routeros.html)
- [DHCP, Option 82](http://www.bubnovd.net/2015/11/dhcp-option-82.html)
- [ Мikrotik RouterOS Configuration Management с помощью скриптов и какой-то там матери ](http://www.bubnovd.net/2017/11/ikrotik-routeros-configuration.html)

## Что изменилось
- С 2020 года я всё реже работаю с сетями и постов о них почти не стало
- Да и вообще с 2020 года в блоге перестали появляться новые статьи. Это связано с тем, что на платформе Blogger неудобно писать - сложно форматировать текст, код выглядит максимально ублюдско, а нормально вставить картинку почти невозможно
- С того времени я периодически задумывался о переезде, но ничего не делал для этого
- Отчасти это связано со сменой направления работы. Из сетевика я окуклился в девопса. А разбираться в новом направлении, да ещё и с важными продакшн задачами и намного более компетентными коллегами, оказалось сложно. Времени и желания на блог просто не оставалось
- В 2017 я завел канал в [Telegram](https://mikrotikninja.t.me). Короткие посты и ссылки стали публиковаться там. За неимением своих статей блог стал не нужен
- А те редкие статьи, которые появлялись, уходили в [Medium](https://medium.com/@dbubnov), [Xakep](https://xakep.ru/author/bubnovd/) и [Telegra.ph](https://telegra.ph/Pik-Uchitel-Ala-Archa-08-12)

## Планы
- [ ] Перенести сюда статьи из [Medium](https://medium.com/@dbubnov), [Xakep](https://xakep.ru/author/bubnovd/) и [Telegra.ph](https://telegra.ph/Pik-Uchitel-Ala-Archa-08-12)
- [ ] Внедрить полезные фичи c [этого](https://digitaldrummerj.me/series/blogging-with-hugo/) ресурса
- [x] Сделать облако тегов
- [ ] Добавить readTime в предпросмотр
- [ ] Добавить related страницы
- [ ] Впилить грамотный CI/CD с линтерами, проверкой орфографии и пережатием и сохранением картинок
- Писать один пост в два месяца. Публиковать его в блоге или на платных платформахх
- Возможно, появится раздел о путешествиях и походах
- Добавить английскую версию блога
- [x] Добавить секцию со ссылками на [blogger](/blogger) на главную
- [x] Добавить верификацию поддомена
- [x] Прикрутить RSS 
- [x] Допилить CI/CD - блог деплоится не всегда, слетает домен
