+++
title = "Система управления конфигурациями Oxidized"
date = 2017-10-12T09:16:00Z
updated = 2017-11-15T00:39:58Z
tags = ["oxidized", "D-Link", "Mikrotik"]
blogimport = true 
aliases = ["/2017/10/oxidized.html"]
[author]
	name = "devi1"
	uri = "https://www.blogger.com/profile/05777499482649623616"
+++

Резервное копирование  - важная составляющая стабильной работы инфраструктуры. Каждый подходит к вопросу резервного копирования со своими взглядами. Сетевое оборудование также нуждается в постоянном бэкапе конфигураций. Но в силу того, что сети редко бывают моновендорными и найти один метод управления для всей сети часто невозможно,  резервное копирование сетевых устройств часто вводит в замешательство администраторов.

  

Для автоматизации резервного копирования конфигураций сетевого оборудования инженеры часто применяют системы контроля версий. Такой подход позволяет не только всегда иметь актуальную копию конфигурации, но и оперативно находить изменения в настройках, которые могли привести к неправильной работе сети.

  

Во время подготовки к выступлению на [MUM](https://mum.mikrotik.com/presentations/RU17M/presentation_4655_1508211463.pdf) [2017](https://www.youtube.com/watch?v=9Ceti-EQpU4) я изучил несколько систем, позволяющих автоматизировать резервное копирование в git.  

  
  
  
## Unimus

  

Разработана Tomas Kirnak - моим коллегой - тренером Mikrotik из США. Система писалась исходя из потребностей Томаса для его сети из 1500 RouterOS и других устройств. На тему управления этим парком Томас [докладывал на МУМе.](https://mum.mikrotik.com/presentations/US17/presentation_4529_1496842532.pdf)

  

[Unimus](https://unimus.net/) инсталлируется на Windows, \*nix, имеется  portable версия для Windows. Устанавливать я не пробовал, но portable версию запустил. Все заработало без особых телодвижений, открылся приятный веб интерфейс. Поддерживается работа с 58 вендорами, бэкапит в git

  
![Картинки по запросу unimus backup](https://unimus.net/images/features/2.gif)  
  

Система простая, удобная и, главное, работающая. За что автор и просит денег.

  
  
![](https://lh6.googleusercontent.com/vz8EnqRnYlbPMLv2nNfHAG_W75lAlXyZQu7pFM8Fuw2XfIjdjE7LxUy1f65zJ17L07qw9pDfimm6xtbte03tLGVcdB9uiiE7LpRhBELWl25eOht-_iFcWBXE6H7CGLs3VXOeaMCR668)  
  
  

## Oxidized

  

Аналогичная предыдущей система. Менее приятный интерфейс, что не мешает выполнять oxidized свою основную функцию. Поддерживается 91 вендор, среди которых Cisco, Juniper, Mikrotik, D-Link, Cumulus Linux, pfSense.

  

На русском языке информацию об oxidized я не нашел. Поэтому постараюсь восполнить этот пробел.

  
  
  

[Oxidized](https://github.com/ytti/oxidized)  создавался как альтернатива RANCID двумя разработчиками: [Saku Ytti](mailto:saku@ytti.fi) и [Samer Abdel-Hafez](mailto:sam@arahant.net). Кстати, парням нужен разработчик на Ruby для помощи в разработке. Явное преимущество над RANCID - удобный веб-интерфейс, взглянуть на него можно по ссылке: [https://oxidized.arahant.net/nodes](https://oxidized.arahant.net/nodes).

  

![](https://lh5.googleusercontent.com/i4WqEIrZdR5CCY2GXkF7pTAeA3eV2bx_bypxi05WaoQDxlv23UBHzZxRjf9X_SAT4hBdken7fUMJM2Z4dMDczRCKTLakIytFqVR9oaVzNTkMAjTcDWzS1X5FgAQYkASNnHksm1R2)

  

В основном окне можно увидеть список устройств и сравнить их бэкапы. Хорошая фича системы - поиск по конфигурациям в верхнем правом углу. Например, можно найти конфиги всех точек доступа с SSID=bubnovd.net. Веб интерфейс очень быстрый! Работая с более, чем 300 девайсами, я не заметил сколь-нибудь заметных задержек.

  

Провалившись в меню девайса, открывается список различных версий конфига этого устройства.

![](https://lh3.googleusercontent.com/TXmPlQCSYOYAmxOnAm4wXPyOInaaq9pHvNkHojzOkXItsebPm0x3kGgJ2vTVu-zI6QWst5A1mIy5uimpGWFvPXWE23p1dHn0s9iNprGg_W_UnpYr1g3Y1lU8u13-M36ygCJYcTZi)

Каждую версию можно сравнить с другой и увидеть что именно происходило с устройством между этими версиями.

![](https://lh4.googleusercontent.com/8szHMyR4qd4AeCde6ump-XxqECBNRJx6-RWpIEa36czdIqNhd86cHLHIFXar6PvgqnllZaADdI95IMZ-kpFpeHlZiCfcWnIgInbIs8koBh31MP3HzkV_LO6DjXpGgqWogxHoLSlZ)

  

На вкладке stats отображается результат резервного копирования.

![](https://lh5.googleusercontent.com/j0RKpoOYBBqFjRDsom5WakOsJb15ajrwrqgHUvKCOOTaYcVIE_-Mz-P3a2-Nonlng_OpHipdBcLSWvED5dtTy7SxRf0WFVhqDgCmK35bZ51LRvsZ79F9KKCgz5bEpZ-W_RIQI5mH)

Можно увидеть сколько попыток бэкапа были неудачными, сколько времени занимает процесс бэкапа, время последнего удачного и неудачного бэкапов.

  
  
  

### Установка

  

Я опишу установку на Debian. Установку ОС и её первичную настройку описывать не буду, благо всё уже [много](http://www.bubnovd.net/2011/03/debian.html)  [раз](http://www.bubnovd.net/2011/08/linux-server-2-iptables.html)   [сделано](http://www.bubnovd.net/2013/02/lvm.html). Всё делал по [инструкции](https://github.com/ytti/oxidized#debian).

  
```
apt install ruby ruby-dev libsqlite3-dev libssl-dev pkg-config cmake libssh2-1-dev

gem install oxidized oxidized-script oxidized-web
```
  

Установка не вызывает вопросов, всё проходит стандартно. В описании системы не очень понятно что и как делать дальше. Я постараюсь объяснить.

  
  
  

### Конфигурация

  

Запускать oxidized с правами суперпользователя - не самая лучшая идея. Поэтому создадим пользователя oxidized, с правами которого будет запускаться система бэкапа:

  

`useradd oxidized`

  

Создадим для нового пользователя домашний каталог, в котором будут лежать конфигурационные файлы, список устройств и база git:

  

`mkdir /home/oxidized`

  

И дадим права на этот каталог юзеру oxidized:

  

`chown oxidized:oxidized /home/oxidized`

  

Первый запуск создаст все необходимые файлы и каталоги. Не забыли, что запускать надо под специальным пользователем?

  

su - oxidized

oxidized

  

Создался каталог `/home/oxidized/.config/oxidized`, в котором находится всё, что нужно для дальнейшей настройки. Основные настройки системы хранятся в файле config в формате YAML. Он состоит из нескольких частей:

*   Input
    
*   Output
    
*   Source
    
*   Основные настройки
    

  
  
  

Привожу свой конфиг:

  
```
bubnov@oxidized:~$ cat /home/oxidized/.config/oxidized/config

\---
//пользователь и пароль, от которых система будет коннектиться к девайсам
username: oxidized
password: p@ssw0rd
//вендор. Я указывал вендора для каждого устройства в файле router.db. Но можно выставить и здесь
model: junos
//Периодичность снятия бэкапа в секундах
interval: 3600
use\_syslog: false
debug: false
threads: 30
//таймаут сессии. Многие устройства не успевают выгрузить конфигурацию в дефолтные 20 секунд. Приходится увеличивать
timeout: 140
//количество попыток снять бэкап с каждого устройства, после чего считается, что бэкап сделать не удалось
retries: 3
prompt: !ruby/regexp /^(\[\\w.@-\]+\[#>\]\\s?)$/
//IP адрес и порт, на котором будет работать REST API (веб интерфейс по простому)
rest: 127.0.0.1:8888
next\_adds\_job: false
Vars:
//позволяет исключить из бэкапа критичную информацию, такую как snmp-community, ключи шифрования и т.п.
 remove\_secret: true
groups: {}
models: {}
pid: "/home/oxidized/.config/oxidized/pid"
log: "/home/oxidized/.config/oxidized/log"
//тип подключения к управляемым устройствам
input:
 default: ssh
 debug: false
 ssh:
    secure: false
//где хранится конфигурации (git, text, ...) и настройки хранилища
output:
 default: git
 git:
    user: oxidized
    email: admin@bubnovd.net
    repo: "/home/oxidized/.config/oxidized/devices.git"
//откуда берется информация о бэкапящихся девайсах
source:
 default: csv
 csv:
    file: "/home/oxidized/.config/oxidized/router.db"
    delimiter: !ruby/regexp /:/
    map:
     name: 0
     model: 1
     ip: 2
model\_map:
 cisco: ios
 juniper: junos
```
  
  
  

Из важного здесь:

-   Логин и пароль пользователя на устройствах, под именем которого будет делаться бэкап. Я рекомендую создать на всех девайсах специального пользователя с ограниченными правами, позволяющими бэкапить. К примеру, на RouterOS это группа read
    
-   Разделы input, output, source. В них указывается как подключаться к устройствам, где взять их список и куда выгружать бэкап
    
-   Адрес и порт REST API. Чуть позже я объясню, почему он должен быть 127.0.0.1:8888
    

  

В разделе source указано, что информацию о девайсах нужно брать из CSV файла, путь к нему и порядок полей в нем. В моем случае формат такой: `имя\_устройства:модель:ip`

  
  
  

Сам файл router.db:
```
RB128:routeros:192.168.3.128
RB121:routeros:192.168.3.1:oxidized:drugoi\_password
sw11:dlink:192.168.3.11:::2222  
``` 

  

В первой строке указан роутер Mikrotik, о чем говорит поле routeros, с адресом 192.168.3.128 и называться в oxidized он будет RB128. Логин и пароль для подключения будет браться из файла config. Второе устройство RB121 имеет другую учетную запись, параметры которой указаны после IP адреса - логин:пароль. А третий девайс производства D-Link, с логином/паролем из файла config, но SSH на порту 2222.

  

Стоит сказать, что устройства можно группировать по вендору или учетным записям. Об этом можно почитать в [официальной документации](https://github.com/ytti/oxidized/blob/master/docs/Configuration.md).

  
  
  
**Авторизация**  
  
  

На этом этапе уже можно запускать систему и набрав в браузере [http://localhost:8888](http://localhost:8888/) увидеть её интерфейс. Но у oxidized есть большой недостаток: нет аутентификации в системе. То есть любой может открыть в браузере веб-интерфейс и увидеть все конфиги ваших сетевых устройств. Разработчик занимается только системой бэкапа и смежные фичи внедрять пока не планирует.

  

Обойдем этот недостаток с помощью Reverse-proxy.

Ставим nginx:

`apt install nginx`

  

И настраиваем его на работу как реверс прокси:  
  
```
nano etc/nginx/sites-available/default

auth\_basic “Username and Password Required”;
auth\_basic\_user\_file /etc/nginx/.htpasswd;

location / {
  proxy\_set\_header Host $host;
  proxy\_set\_header X-Real-IP $remote\_addr;
  proxy\_set\_header X-Forwarded-For $proxy\_add\_x\_forwarded\_for;
  proxy\_pass http://127.0.0.1:8888;
}
```
  
  

И создаем юзера и пароль:

`sudo htpasswd /etc/nginx/.htpasswd username`

  

Теперь nginx будет работать на стандартном порту (или на том, который вы укажете в его настройках), при обращении к нему будет происходить авторизация и пользователь будет перенаправлен на адрес proxy\_pass (127.0.0.1:8888 в нашем случае).

  
  

Хочу обратить ваше внимание на опцию remove\_secrets в конфиге. Она удаляет критичную информацию из бэкапа, такую как SNMP-community, ключи шифрования, ключи Wi-Fi. Например, Mikrotik RouterOS в дефолте умеет скрывать эти данные, если экспорт выполнять с опцией hide-sensitive. Oxidized же может исключать эти данные из конфигов любого вендора.

Узнать что именно удаляется из конфига, можно посмотрев в описание вашего вендора в [списке](https://github.com/ytti/oxidized/blob/master/lib/oxidized/model/). Например, для Cicso:

![](https://lh5.googleusercontent.com/-WEhbGXGjlGVfV9lp2Sn-BLIPNBUEe2n6npRplCIgHLW8uBAlV2mT7bpABaLhqYshhXin6veWhKWQbzt5cXbCPUwzXHb03JTYGTkguJPyYM4SRZeZBw0QdErHg5C2UZGLWmp9q3W)

  
  
  

Для того, чтобы oxidized стартовал как служба, сделайте следующее:
```
sudo cp /usr/local/share/gems/gems/oxidized-0.19.0/extra/oxidized.service /lib/systemd/system/  
sudo systemctl enable oxidized.service  
sudo systemctl start oxidized
```
  
  
  

Я же просто прописал в crontab запуск после ребута:

```
sudo -u oxidized crontab -l

@reboot /usr/local/bin/oxidized  
```  
  
  

Для подготовки к посту использовались следующие материалы:

[https://github.com/ytti/oxidized#installation](https://github.com/ytti/oxidized#installation)  
[https://unimus.net/](https://unimus.net/)

[https://neckercube.com/index.php/2017/05/04/how-to-install-oxidized-for-network-configuration-backup/](https://neckercube.com/index.php/2017/05/04/how-to-install-oxidized-for-network-configuration-backup/)

[http://packetpushers.net/install-oxidized-network-configuration-backup/](http://packetpushers.net/install-oxidized-network-configuration-backup/)

[http://www.whoopis.com/core/oxidized-quickstart-tutoria.html](http://www.whoopis.com/core/oxidized-quickstart-tutoria.html)

[https://log.cyconet.org/2016/01/29/oxidized-silly-attempt-at-really-awesome-new-cisco-config-differ/](https://log.cyconet.org/2016/01/29/oxidized-silly-attempt-at-really-awesome-new-cisco-config-differ/)

[http://shairosenfeld.blogspot.ru/2011/03/authorization-header-in-nginx-for.html](http://shairosenfeld.blogspot.ru/2011/03/authorization-header-in-nginx-for.html)

[https://community.openhab.org/t/using-nginx-reverse-proxy-authentication-and-https/14542](https://community.openhab.org/t/using-nginx-reverse-proxy-authentication-and-https/14542)

  
  
  

Надеюсь, эта статья поможет вам в настройке резервного копирования. Любые вопросы вы можете задать в комментариях к статье. Я постараюсь на них ответить.