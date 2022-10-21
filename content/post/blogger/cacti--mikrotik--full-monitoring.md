+++
title = "Cacti + Mikrotik = Full Monitoring"
date = 2014-03-01T02:21:00Z
updated = 2014-03-01T03:17:15Z
tags = ["cacti", "SNMPv3", "Mikrotik", "monitoring"]
aliases = ["/2014/03/cacti-mikrotik-full-monitoring.html"]
blogimport = true 
[author]
	name = "devi1"
	uri = "https://www.blogger.com/profile/05777499482649623616"
+++

  
Захотелось мне собирать все логи с сетевого оборудования в одном месте. Решил поднять syslog-сервер. Так как с такими сервисами никогда не сталкивался, пришлось спросить у великого гугля. Он подсказал мне, что с этим неплохо справляется система мониторинга Cacti. А о ней то я уже наслышан, и в планах стояло внедрение сего чуда в сеть. Итак, теперь основная цель не сислог, а мониторинг - то, для чего и создан cacti. Настроим мониторинг, а потом примемся за сислог.  
  
Система мониторинга Cacti разрабатывалась с целью визуализации данных, поэтому сравнивать её с другими будет несколько неуместно. Я для мониторинга активности устройств использую The Dude. И советую её всем, у кого сеть построена на микротиках, т.к. разработана она этой компанией, и, соответственно, умеет полностью управлять Mikrotik'ом. Но об этом в следующей статье.  
  
Итак, cacti можно установить на Windows и UNIX-like системы. Для работы ей нужны:  

1.  web-server (apache 2.2)
2.  database server (mySQL 5.6)
3.  php (php 5.4)
4.  RRDTool (1.4.8)
5.  net-snmp (5.7)

В скобках указаны версии, установленные мной на FreeBSD 10.0 (запущенной под Hyper-V в Windows 2012). Процесс установки подробно расписывать не буду. Я руководствовался этими ресурсами:  

1.  [Установка Cacti (краткая заметка)](http://nix-sa.blogspot.ru/2011/08/cacti.html)
2.  [Кактус у монитора или ускоренная установка cacti](http://habrahabr.ru/post/71087/)
3.  [Cacti – простой и удобный инструмент для мониторинга и анализа сети](http://samag.ru/archive/article/1770) 

 Я давно не работал с UNIX-like системами, поэтому возникли некоторые проблемы (будьте осторожны выбирая пакеты, которые тянет php. Он может вытянуть иксы). Для тех, кто впервые ставит LAMP, советую почитать мануалы по установке и конфигурированию сервисов.  
  
Ставим [плагин](http://docs.cacti.net/plugin:mikrotik) для работы с Mikrotik (копируем папку в /usr/share/local/cacti/plugins и идем в веб-интерфейсе кактуса в Configuration - Plugin Management и нажимаем Install, Activate). При его установке выпадает ошибка. Лечится удалением строки 'Invalid round robin archive' из /usr/local/share/cacti/plugins/mikrotik/templates/cacti\_host\_template\_mikrotik\_device.xml.  
  
Настраиваем SNMP на Mikrotik. IP -> SNMP -> Communities. Использовать стандартный public не рекомендую, т.к. он передаёт данные в открытом виде и без авторизации. Мы будем использовать SNMP v3, поддерживающий авторизацию и шифрование. Добавляем новую community:  

*   имя - любое (cacti)
*   Addresses - адрес сервера SNMP (на котором стоит cacti)
*   Security - authorized
*   Read/write - выбираем по своему усмотрению, read нужен обязательно, а write только если планируем конфигурировать роутер по SNMP протоколу
*   Протоколы авторизации и шифрования оставляем дефолтные
*   Задаём пароль авторизации и парольную фразу для шифрования (могут быть разные значения)

  

[![](http://3.bp.blogspot.com/-qeSZdaw9AGc/UxGx8cC4McI/AAAAAAAAAe0/Ck3khsntuKo/s1600/mt-snmp.png)](http://3.bp.blogspot.com/-qeSZdaw9AGc/UxGx8cC4McI/AAAAAAAAAe0/Ck3khsntuKo/s1600/mt-snmp.png)

  
В SNMP Settings пишем:  

*   Contact Info - почта админа
*   Location - физическое расположение устройства
*   Trap Target - адрес SNMP сервера
*   Выбираем созданную нами Trap Community
*   Trap Version - 3
*   Trap Generation - кто будет генерировать сообщения для сервера - интерфейс или встроенный сервер
*   Выбираем нужные интерфейсы

  
Добавляем в cacti наш микротик. Подробно распишу только про SNMP options:  

*   SNMP Version - 3
*   SNMP Username (v3) -  имя нашего Community (cacti)
*   SNMP Password (v3) - то, что ввели в Authentication Password в Mikrotik
*   SNMP Privacy Passphrase (v3) - Encryption Password в Mikrotik

По дальнейшей работе с кактусом полно мануалов. Также есть отличная документация на оф. сайте, правда на английском. [Презентация](http://mum.mikrotik.com/presentations/CZ09/schaub.pdf) о настройке SNMP на Mikrotik.  
  
Первый час работы кактуса:  

[![](http://2.bp.blogspot.com/-Nmv5KEBWAUs/UxG0IYWDldI/AAAAAAAAAfA/rGKLw1ncLhA/s1600/Screenshot+from+2014-03-01+16:18:55.png)](http://2.bp.blogspot.com/-Nmv5KEBWAUs/UxG0IYWDldI/AAAAAAAAAfA/rGKLw1ncLhA/s1600/Screenshot+from+2014-03-01+16:18:55.png)

За сим откланяюсь. Постараюсь сочинить что-нибудь про The Dude. Да и про кактус, когда сам в нем разберусь.