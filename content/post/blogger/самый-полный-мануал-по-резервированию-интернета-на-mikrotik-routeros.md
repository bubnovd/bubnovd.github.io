+++
title = "Самый полный мануал по резервированию интернета на Mikrotik RouterOS"
date = 2015-03-05T22:06:00Z
updated = 2017-07-26T23:24:47Z
tags = ["routeros", "Mikrotik", "script", "failover"]
aliases = ["/2015/03/mikrotik-routeros.html"]
blogimport = true 
[author]
	name = "devi1"
	uri = "https://www.blogger.com/profile/05777499482649623616"
+++

В сети много мануалов по фэйловеру на RouterOS и подобрать нужный под конкретные цели иногда проблематично. В ходе своих экспериментов я выяснил, что наиболее универсальным является [этот](http://wiki.mikrotik.com/wiki/Advanced_Routing_Failover_without_Scripting) способ. Но у меня возникло несколько вопросов, ответы на которые я пока не нашел.  
  
```
/ip route  
add dst-address=**Host1** gateway=GW1 scope=**10**  
add dst-address=**Host2** gateway=GW2 scope=**10**
```
  
```
/ip route  
add distance=1 gateway=**Host1** routing-mark=ISP1 check-gateway=ping  
add distance=2 gateway=**Host2** routing-mark=ISP1 check-gateway=ping
```
  
```
/ip route  
add distance=1 gateway=**Host2** routing-mark=ISP2 check-gateway=ping  
add distance=2 gateway=**Host1** routing-mark=ISP2 check-gateway=ping
```
  
  
Я делаю немного по-другому: первые две строки точно такие же, а дальше:  
```
/ip route  
add distance=1 gateway=**Host1** check-gateway=ping  
add distance=2 gateway=**Host2** check-gateway=ping
```
  
```
/ip route  
add distance=1 gateway=**GW1** routing-mark=ISP1  
add distance=1 gateway=GW**2** routing-mark=ISP2
```
  
В первом варианте мне непонятно зачем пускать трафик, маркированный ISP1 по второму каналу и аналогично с ISP2. Чтобы не рвалась сессия? Она все равно рвется. В общем, я делаю по своему варианту.  
  
Как оно работает?  
`add dst-address=Host1 gateway=GW1 scope=10` указываем, что трафик до Host1 будет ходить через провайдера ISP1. Аналогично со вторым.  
  
`add distance=1 gateway=Host1 check-gateway=ping` указывает шлюзом по умолчанию Host1. Тем самым создается рекурсивный маршрут через GW1. Проверяется доступность Host1 пингом. Если Host1 доступен, то шлюз - GW1, если не доступен, но доступен Host2, трафик идет через GW2.  
  
Маркировка маршрутов необходима для того, чтобы трафик, пришедший в один из интерфейсов не улетел во второй, если метрика второго ниже. Маркировка создается в таблице Mangle фаервола:  
```
add chain=prerouting in-interface=ISP1-int action=mark-connection new-connection-mark=ISP1 passthrough=yes   
add chain=prerouting connection-mark=ISP1 action=mark-routing new-routing-mark=ISP1  
add chain=output connection-mark=ISP1 action=mark-routing new-routing-mark=ISP1  
```
Так же по второму провайдеру.  
  
Эта схема прекрасно работает при статическом IP. Но как быть с DHCP, PPP и т.д. А всё просто - в настройках DHCP/PPP-клиента отключаем Add Default Route. И добавляем необходимый маршрут руками. Единственная проблема - адрес шлюза может смениться. В таком случае необходимо применение скриптов. Я написал небольшой скрипт для работы с USB-модемом, который проверяет правильный ли шлюз указан в строке `8.8.4.4 gw GW2`. PPP-интерфейс в моем случае называется `modem`.  
```
:local testIP 8.8.4.4/32  
:local modemGW \[/ip address get \[find interface="modem"\] network\]  
if ($modemGW!=(\[/ip route get \[find dst-address=$testIP\] gateway\])) do={  
\[/ip route set \[find dst-address=$testIP\] gateway $modemGW\]}  
```
В Scheduler добавляем расписание для этого скрипта, чтобы выполнялся раз в минуту. На основе моего скрипта можно создать свой для любого другого типа подключения.  
Для нетерпеливых привожу полный список команд, необходимый для настройки модема в качестве резервного канала. Всё, что вам нужно - просто скопировать текст и вставить его в терминал. Основной придется отмаркировать и прописать в маршрутизации руками:  
```
/ip firewall mangle  
add chain=input in-interface=modem action=mark-connection new-connection-mark=modem  
add chain=output connection-mark=modem action=mark-routing new-routing-mark=modem  
/ip route  
add dst-address=8.8.4.4/32 gateway=\[/ip address get \[find interface="modem"\] network\] distance=1 scope=10  
add distance=2 gateway=8.8.4.4 scope=10 check-gateway=ping  
add dst-address=0.0.0.0/0 gateway=\[/ip address get \[find interface="modem"\] network\] distance=1 scope=10 routing-mark=modem  
/interface ppp-client set modem dial-on-demand=no add-default-route=no  
/system script  
add name=modemGW source=":local testIP 8.8.4.4/32\\r\\n:local modemGW \[/ip address get \[find interface=\\"modem\\"\] network\]\\r\\nif (\\$modemGW!=(\[/ip route get \[find dst-address=\\$testIP\] gateway\])) do={\\r\\n\[/ip route set \[find dst-address=\\$testIP\] gateway \\$modemGW\]}"  
/system scheduler  
add name=checkGW interval=00:01:00 on-event=modemGW  
```
  
Есть ещё один способ фэйловера. Делается он только с помощью скриптов. Полное его описание [здесь](http://habrahabr.ru/post/141785/). Ниже скрипт для его мгновенного создания:  
```
/system script  
add name=SetGlobalParameters source="#Main interface name\\r\\n:global MainIf \[/interface get \[find comment~\\"MainINT\\"\] name\]\\r\\n#Reserve interface name\\r\\n:global RsrvIf \[/interface get \[find comment~\\"RsrvINT\\"\] name\]\\r\\n#Main interface ip address\\r\\n:global MainIfAddress \\"\\"\\r\\n#Reserve interface ip address\\r\\n:global RsrvIfAddress \\"\\""  
add name=DefineMainIfIp source=":global MainIf\\r\\n:global MainIfAddress \\"\\"\\r\\n:set MainIfAddress \[/ip address get \[find interface=$MainIf\] address\]"  
add name=DefineReservedIfIp source=":global RsrvIf\\r\\n:global RsrvIfAddress \\"\\"\\r\\n:set RsrvIfAddress \[/ip address get \[find interface=$RsrvIf\] address\]"  
add name=ConnectionCheck source=":global MainIf\\r\\n:global RsrvIf\\r\\n:global MainIfAddress\\r\\n:global RsrvIfAddress\\r\\n\\r\\n:local PingCount 3\\r\\n\\r\\n#yandex DNS\\r\\n:local PingTarget1 77.88.8.8\\r\\n\\r\\n#OpenDNS\\r\\n:local PingTarget2 208.67.222.222\\r\\n\\r\\n#google dns\\r\\n:local PingTarget3 8.8.8.8\\r\\n\\r\\n#Check main internet connection\\r\\n\\r\\n:local MainIfInetOk false;\\r\\n\\r\\nif (\\$MainIfAddress=\\"\\") do={delay 5}\\r\\n\\r\\nif (\\$MainIfAddress!=\\"\\") do={\\r\\n\\r\\n:local PingResult1 \[/ping \\$PingTarget1 count=\\$PingCount interface=\\$MainIf\]\\r\\n:local PingResult2 \[/ping \\$PingTarget2 count=\\$PingCount interface=\\$MainIf\]\\r\\n:local PingResult3 \[/ping \\$PingTarget3 count=\\$PingCount interface=\\$MainIf\]\\r\\n:set MainIfInetOk ((\\$PingResult1 + \\$PingResult2 + \\$PingResult3) >= (2 \* \\$PingCount))\\r\\n}\\r\\n\\r\\n#Check reserved internet connection\\r\\n:local RsrvIfInetOk false;\\r\\n\\r\\nif (\\$RsrvIfAddress=\\"\\") do={delay 5}\\r\\n\\r\\nif (\\$RsrvIfAddress!=\\"\\") do={\\r\\n:local PingResult1 \[/ping \\$PingTarget1 count=\\$PingCount interface=\\$RsrvIf\]\\r\\n\\r\\n:local PingResult2 \[/ping \\$PingTarget2 count=\\$PingCount interface=\\$RsrvIf\]\\r\\n:local PingResult3 \[/ping \\$PingTarget3 count=\\$PingCount interface=\\$RsrvIf\]\\r\\n\\r\\n:set RsrvIfInetOk ((\\$PingResult1 + \\$PingResult2 + \\$PingResult3) >=(2 \* \\$PingCount))\\r\\n}\\r\\n\\r\\n:put \\"MainIfInetOk=\\$MainIfInetOk\\"\\r\\n:put \\"RsrvIfInetOk=\\$RsrvIfInetOk\\"\\r\\n\\r\\nif (!\\$MainIfInetOk) do={\\r\\n/log error \\"Main internet connection error\\"\\r\\n}\\r\\n\\r\\nif (!\\$RsrvIfInetOk) do={\\r\\n/log error \\"Reserve internet connection error\\"\\r\\n}\\r\\n\\r\\n:local MainGWDistance \[/ip route get \[find comment~\\"MainGW\\"\] distance\]\\r\\n:local RsrvGWDistance \[/ip route get \[find comment~\\"RsrvGW\\"\] distance\]\\r\\n:put \\"MainGWDistance=\\$MainGWDistance\\"\\r\\n:put \\"RsrvGWDistance=\\$RsrvGWDistance\\"\\r\\n\\r\\n#SetUp gateways\\r\\nif (\\$MainIfInetOk && (\\$MainGWDistance >= \\$RsrvGWDistance)) do={\\r\\n/ip route set \[find comment~\\"MainGW\\"\] distance=1\\r\\n/ip route set \[find comment~\\"RsrvGW\\"\] distance=2\\r\\n/log info \\"Switch to main internet connection\\"\\r\\n}\\r\\n\\r\\nif (!\\$MainIfInetOk && \\$RsrvIfInetOk && (\\$MainGWDistance <= \\$RsrvGWDistance)) do={\\r\\n/ip route set \[find comment~\\"MainGW\\"\] distance=2\\r\\n/ip route set \[find comment~\\"RsrvGW\\"\] distance=1\\r\\n/log warning \\"Switch to reserve internet connection\\"\\r\\n}"  
run SetGlobalParameters  
/system scheduler  
add name=SetGlobalParameters start-time=startup on-event=SetGlobalParameters  
add name=DefineMainIfIp interval=00:00:27 on-event=DefineMainIfIp  
add name=DefineReservedIfIp interval=00:00:27 on-event=DefineReservedIfIp  
add name=ConnectionCheck interval=00:01:00 on-event=ConnectionCheck
```