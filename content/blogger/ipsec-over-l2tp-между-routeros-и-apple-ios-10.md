+++
title = "IPSec over L2TP между RouterOS и Apple iOS 10"
date = 2016-10-10T03:23:00Z
updated = 2017-10-16T04:45:32Z
tags = ["ios10", "apple", "ios", "routeros", "l2tp", "Mikrotik", "iphone", "ipsec"]
blogimport = true 
aliases = ["/2016/10/ipsec-over-l2tp-routeros-apple-ios-10.html"]
[author]
	name = "devi1"
	uri = "https://www.blogger.com/profile/05777499482649623616"
+++
В 10 версии iOS команда Apple наконец-то выпилила PPTP, чем сподвигла весь (не)цивилизованный ИТ-мир срочно учиться поднимать IPSec на своих бордерах. В том числе и меня.  
  
"Поднять IPSec - что там сложного" - подумал я. Но не тут то было. По мануалам, хабрам и прочим ресурсам все отлично поднимается и работает между двумя микротиками, между микротиком и виндой, между микротиком и андроид. Но вот с iOS 10 ни в какую не хочет. Путем долгих ковыряний и трехэтажных словосочетаний, выяснил, что изменения для IPSec нужно применять в КОНСОЛИ! Не в графическом интерфейсе, а именно в консоли - ssh или встроенная в WinBox - без разницы. Но факт в том, что идентичные настройки в WinBox не позволяли поднять VPN между Mikrotik RouterOS и IPhone (по крайней мере в моем случае с RB751 на RouterOS 6.37.1 в связке с IPhone 6).  
  
  
  
Привожу настройки L2TP и IPSec текстом и скрины из WinBox что должно получиться:  
  
L2TP сервер  
```
/interface l2tp-server server  
set authentication=mschap2 default-profile=default enabled=yes  
```  

[![](https://2.bp.blogspot.com/-_4MPqiOY7dM/V_tkhkfGTGI/AAAAAAAAAxc/iUs3UlAZQCMHEWNAoQqQbCrCU7hH586CQCLcB/s320/1_l2tp.PNG)](https://2.bp.blogspot.com/-_4MPqiOY7dM/V_tkhkfGTGI/AAAAAAAAAxc/iUs3UlAZQCMHEWNAoQqQbCrCU7hH586CQCLcB/s1600/1_l2tp.PNG)

L2TP server

  
PPP профили дефолтные  
```
\> ppp profile print   
Flags: \* - default   
 0 \* name="default" use-mpls=default use-compression=default   
     use-encryption=default only-one=default change-tcp-mss=yes   
     use-upnp=default address-list="" on-up="" on-down=""   
  
 1 \* name="default-encryption" use-mpls=default use-compression=default   
     use-encryption=yes only-one=default change-tcp-mss=yes use-upnp=default   
     address-list="" on-up="" on-down=""  
```  

[![](https://2.bp.blogspot.com/-4j6jmkS_1zM/V_tlNc9yazI/AAAAAAAAAxk/nnkmtuOYXDQtuOicW1Gb4wKXzSNpe_ohwCLcB/s320/2_ppp-profile.PNG)](https://2.bp.blogspot.com/-4j6jmkS_1zM/V_tlNc9yazI/AAAAAAAAAxk/nnkmtuOYXDQtuOicW1Gb4wKXzSNpe_ohwCLcB/s1600/2_ppp-profile.PNG)

ppp profile

  
Пользователь L2TP тоже ничем не отличается от простого L2TP/PPTP и иже с ними без айписека  
```
/ppp secret

add local-address=192.168.222.16 name=user1 password=l2tppassword profile=default-encryption remote-address=192.168.222.18 service=l2tp
```
[![](https://2.bp.blogspot.com/-ruLmJPDInsM/V_tl5BDgwQI/AAAAAAAAAxo/jYPxZzvOKTEO7ZPh7XGp7jQpwTOLlvrfwCLcB/s320/3_user.PNG)](https://2.bp.blogspot.com/-ruLmJPDInsM/V_tl5BDgwQI/AAAAAAAAAxo/jYPxZzvOKTEO7ZPh7XGp7jQpwTOLlvrfwCLcB/s1600/3_user.PNG)

L2TP secret

  

  
  
  
Приступаем к IPSec. Все, что я делал с IPSec - делал через консоль. Из винбокса не заработало. что именно сделалось не так - не проверял. Можете проверить методом исключения.  
  
Добавляем группу (дефолтная работает криво - особенность RouterOS):  
```
/ip ipsec policy group  
add name=l2tp
```
[![](https://3.bp.blogspot.com/-J0qbeMyCGsE/V_tm3ZmyyGI/AAAAAAAAAx4/qkVDbLAHMnYiRGtyTh4gHMbAbRXwmClHgCLcB/s320/4_group.PNG)](https://3.bp.blogspot.com/-J0qbeMyCGsE/V_tm3ZmyyGI/AAAAAAAAAx4/qkVDbLAHMnYiRGtyTh4gHMbAbRXwmClHgCLcB/s1600/4_group.PNG)

IPSec group

  

  

Корректируем proposal и добавляем новый:
```
/ip ipsec proposal
set \[ find default=yes \] enc-algorithms=aes-256-cbc,aes-128-cbc
add enc-algorithms=aes-256-cbc,aes-128-cbc name=L2TP pfs-group=none
```
  

[![](https://4.bp.blogspot.com/-V5leQjYiI90/V_tnS-HkCRI/AAAAAAAAAx8/Es4LO8wuR8cMHJoDJJz3JKpaR7JprFvzwCLcB/s320/5_proposal.PNG)](https://4.bp.blogspot.com/-V5leQjYiI90/V_tnS-HkCRI/AAAAAAAAAx8/Es4LO8wuR8cMHJoDJJz3JKpaR7JprFvzwCLcB/s1600/5_proposal.PNG)

IPSec proposal

  
Добавляем пира:  
```
/ip ipsec peer
add enc-algorithm=aes-256,aes-192,aes-128,3des exchange-mode=main-l2tp generate-policy=port-strict passive=yes policy-template-group=l2tp secret=RouterOS
```
  

[![](https://4.bp.blogspot.com/-g2hwHl6Dboc/V_toMc017GI/AAAAAAAAAyI/Ml-pWpj0oXQ5J-BZ99i7PI0TsvjKX1FWgCLcB/s320/6_peer.PNG)](https://4.bp.blogspot.com/-g2hwHl6Dboc/V_toMc017GI/AAAAAAAAAyI/Ml-pWpj0oXQ5J-BZ99i7PI0TsvjKX1FWgCLcB/s1600/6_peer.PNG)

IPSec peer

  
  

Добавляем шаблон политики:
```
/ip ipsec policy
add dst-address=0.0.0.0/0 group=l2tp proposal=L2TP src-address=0.0.0.0/0 template=yes
```
[![](https://1.bp.blogspot.com/-BREipqX5fhY/V_toquFtKXI/AAAAAAAAAyQ/I5njzVpQlLcE8jdqW6IrYTJVAWT8zN6VwCLcB/s320/7_policy.PNG)](https://1.bp.blogspot.com/-BREipqX5fhY/V_toquFtKXI/AAAAAAAAAyQ/I5njzVpQlLcE8jdqW6IrYTJVAWT8zN6VwCLcB/s1600/7_policy.PNG)

IPSec policy

  

[![](https://2.bp.blogspot.com/-KrcYxMQcPgM/V_toqs6DMsI/AAAAAAAAAyM/-48BTpkfmbE4XnCjflmwOwE-7lAGOcVgQCLcB/s320/8_policy.PNG)](https://2.bp.blogspot.com/-KrcYxMQcPgM/V_toqs6DMsI/AAAAAAAAAyM/-48BTpkfmbE4XnCjflmwOwE-7lAGOcVgQCLcB/s1600/8_policy.PNG)

IPSec policy action

  

  

  

Ну и настройка IPhone:

[![](https://2.bp.blogspot.com/-XZYGv6-IFL4/V_trrzjAp5I/AAAAAAAAAyg/0tyDmiUx3NEBF9J8E3ns3yLX6Se5hrlFwCLcB/s320/0_iphone.PNG)](https://2.bp.blogspot.com/-XZYGv6-IFL4/V_trrzjAp5I/AAAAAAAAAyg/0tyDmiUx3NEBF9J8E3ns3yLX6Se5hrlFwCLcB/s1600/0_iphone.PNG)

IPhone

  

  

Спасибо [Кириллу Васильеву](https://www.vasilevkirill.com/) за наводку  
  
UPD: если имеются проблемы с подключением, попробуйте в /ip ipsec peer generate-policy установить port-override вместо port-strict 

  
UPD: как советуют в комментариях  

[Евгений](https://vk.com/eugene_fesko)[15.10.2017, 1:17](http://www.bubnovd.net/2016/10/ipsec-over-l2tp-routeros-apple-ios-10.html?showComment=1508055431990#c7669898995946448842)

В общем как ни странно вопрос решён, действительно помогло. А значит тем, кто кроме меня столкнулся с такой же проблемой, но не хочет прошиваться на релиз-кандидаты прошивок и ждёт официальную - им поможет отключение UPNP в роутере (в принципе его есть смысл юзать в домашнем варианте использования роутера, но и по работе в некоторых случаях он бывает полезен, в таком случае решение - только прошиваться на 6.41rc38)