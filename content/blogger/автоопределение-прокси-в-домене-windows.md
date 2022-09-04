+++
title = "Автоопределение прокси в домене Windows"
date = 2013-09-13T04:39:00Z
updated = 2016-08-08T03:01:10Z
tags = ["DNS", "windows", "WPAD", "proxy"]
aliases = ["/2013/09/windows.html"]
blogimport = true 
[author]
	name = "devi1"
	uri = "https://www.blogger.com/profile/05777499482649623616"
+++

Автоматизация - дело хорошее. И автоматизирование всего и вся поможет улучшить жизнь системного администратора если не в текущий момент, то в обозримом будущем точно. Для этого решил я настроить автоконфигурирование настроек прокси-сервера на клиентских машинах.  
Имеется апдейт!  
  
Как прокси мы используем Usergate (будь он неладен. На человеческие прокси пока не могу уломать начальство). В нем есть свой DHCP сервер с нужной нам функцией. Но в качестве DHCP мы используем стандартные средства Windows Server, поэтому простой вариант с одной галочкой нам не подходит.  
 Тут нам на помощь приходит протокол автонастройки прокси ([WPAD](https://en.wikipedia.org/wiki/Web_Proxy_Autodiscovery_Protocol)). Смысл его работы в следующем:  

1.  Клиент ищет в сети сервер с именем wpad.domain.local (domain.local тут - имя нашего домена), если такой сервер не находится клиент идет выше - wpad.local (в этом кстати заключается брешь этого протокола, о которой говорили на одной из blackhat-конференций).
2.  Если сервер найден, клиент пытается прочитать файл wpad.dat, лежащий в корневом каталоге.
3.  Из этого файла берутся настройки, исходя из которых клиент понимает, в каких случаях нужно идти в сеть через прокси, а в каких напрямую.

Пример простейшего файла wpad.dat: 
``` 
function FindProxyForURL(url, host)  
{  
    return "PROXY 192.168.0.254:3128";  
}  
```
Здесь мы видим, что в качестве адреса прокси-сервера клиенты получают `192.168.0.254:3128`. Настройки можно кастомизировать, у файла может быть огромное количество опций. Их вы можете найти в [поисковике](http://www.google.ru/search?q=wpad+example).  
  
Итак, будем считать, что файл настроек создан. Нужно положить его в корневую директорию веб-сервера и настроить сам веб-сервер, чтобы он умел отдавать этот файл. У нас используется Apache под виндой, поэтому привожу его конфигурацию: в httpd.conf в секции `<IfModule mime\_module>` нужно добавить строку `AddType application/x-ns-proxy-autoconfig .dat`
  
Следующий этап - настройка DNS-сервера. Нужно внести запись типа CNAME в зону прямого просмотра, указывающую, что под именем `wpad.domain.local` будет пониматься ваш веб-сервер. На этом натыкаемся на проблему - винда не разрешает использовать имя wpad. Об этом вам ничего не скажет, но найти сервер по имени wpad в сети вы не сможете. Нужно выполнить в командной строке `dnscmd /config /enableglobalqueryblocklist 0` (подробнее [здесь](http://technet.microsoft.com/ru-ru/library/cc441517.aspx)).  
  
Очищаем кэш ДНС или просто идем курить пить кофе. Выставляем в браузере автонастройку прокси-сервера и получаем счастье. Теперь, если изменится адрес или порт прокси, или просто добавится сервер, на который нужно ходить в обход прокси, то настройки делаются в одном месте, а до клиентов доходят сами.  
  
Спасибо [ит-бложику](http://it-blojek.ru/19/) за наводку.  
  
UPD 24.12.2013  
Описание параметров в файле wpad.dat производится на языке JavaScript, т.е. можно построить довольно сложную конфигурацию. Вот пример моего файла, проверяющего соответствие IP-адресов и закомментированными сообщениями для дебага (строки, начинающиеся с var и ниже проверяют, находится ли адрес в диапазоне 192.168.100.65 - 192.168.100.240):  
```  
function FindProxyForURL(url, host)  
{  
// If specific URL needs to bypass proxy, send traffic direct.  
if (shExpMatch(host,"\*subd.domain.local\*") ||  
shExpMatch(host,"\*subd.domain1.local\*") ||  
shExpMatch(host,"192.168.\*") ||  
shExpMatch(host,"127.\*") ||  
dnsDomainIs(host,".subd.domain.local") ||  
dnsDomainIs(host,".subd.domain1.local") ||  
isPlainHostName(host))  
return "DIRECT";  
  
// All other traffic uses below proxies, in fail-over order.  
//debug="host=" + host + "\\n" + "url=" + url + "\\n" + "MyIP=" + myIpAddress() +  "\\n";  
var myip = myIpAddress();  // Set a variable for the local IP address.  
var myAddrArray = myip.split(".");  // Split that IP address var into an array.  
var mysubnet3=parseInt(myAddrArray\[2\]);  // Convert array element #2 into a number and store it in a new variable  
var mysubnet4=parseInt(myAddrArray\[3\]);  
//debug += "Subnet=" + mysubnet3 + "." + mysubnet4 + "\\n";  
if ((mysubnet3 == 100) && (mysubnet4 >=65) && (mysubnet4 <= 240)) {  
return "PROXY 192.168.100.14:8080";}  
else {  
return "DIRECT";}  
}  
```  
В качестве веб-сервера для этой системы очень удобно использовать `tinyhttpd`, если нет уже работающих серверов. tynihttpd очень маленький и удобный, не нужно делать ничего лишнего - только `index.html`, `wpad.dat` и запустить экзешник из cmd.  
Ещё DNS сервер Windows не дает разрешить доменные имена wpad и isatap, поэтому с ним нужно поколдовать: в командной строке пишем `dnscmd /config /globalqueryblocklist` и будет счастье.  
  
`**UPD 01.09.2014**`  
`**[Здесь](https://securelist.ru/analysis/obzor/242/pac-fajl-avtokonfiguratsii-problem/) интересная статейка по этой теме.**`  
`**UPD 08.08.2016**`  
**[О безопасности при использовании WPAD](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/pbp-final-with-update.pdf)**