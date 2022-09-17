+++
title = "Yet another RouterOS script"
date = 2015-12-09T22:41:00Z
updated = 2015-12-10T01:00:22Z
tags = ["routeros", "Mikrotik", "python", "script", "rsc"]
blogimport = true 
[author]
	name = "devi1"
	uri = "https://www.blogger.com/profile/05777499482649623616"
+++

Скрипт ищет интерфейс sstp клиента, имя которого содержит в себе строку video, копирует имя интерфейса, имя пользователя и пароль для подключения, а так же адрес vpn сервера. Удаляет упоминание об этом интерфейсе в настройках OSPF (т.к. после удаления появится интерфейс unknown, а без удаления интерфейса новый не создастся - имена одинаковые), создает нешифрованное соединение pptp с аналогичными настройками и вписывает его в параметры OSPF.<br /><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;">#!Ищет sstp-интерфейс с именем, содержащим video</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;">#! копирует его настройки, убирает номер порта и</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;">#! создает pptp интерфейс с подобными настройками</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;">#! Copyright Dmitry Bubnov http://bubnovd.net</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><br /></span><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;">/interface sstp-client</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;">:local name [get [find name~"video"] name]</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;">:local srv [get [find name~"video"] connect-to]</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;">:local conto [:pick $srv 0 ([:len $srv]-4)]</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;">:local user [get [find name~"video"] user]</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;">:local pwd [get [find name~"video"] password]</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;">/routing ospf interface remove [find interface=$name]</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;">remove $name</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><br /></span><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;">/interface pptp-client add connect-to=$conto user=$user password=$pwd name=$name allow=mschap2 disabled=no profile=default</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;">/routing ospf interface add interface=$name cost=9 network-type=point-to-point&nbsp;</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"></span><br /><a name='more'></a><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><span style="font-family: &quot;georgia&quot; , &quot;times new roman&quot; , serif;">Он же в Python для пакетного изменения:</span></span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><br /></span><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;">#!/usr/bin/env python</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><br /></span><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;">from RosAPI import Core</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><br /></span><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><br /></span><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;">mas = []</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;">with open("list.txt") as f:</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><span class="Apple-tab-span" style="white-space: pre;"> </span>mas = f.read().splitlines()</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><br /></span><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;">for i in range(len(mas)):</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><span class="Apple-tab-span" style="white-space: pre;"> </span>try:</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><span class="Apple-tab-span" style="white-space: pre;">  </span>a = Core(mas[i])</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><br /></span><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><span class="Apple-tab-span" style="white-space: pre;"> </span>except:</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><span class="Apple-tab-span" style="white-space: pre;">  </span>print "No Connection"</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><span class="Apple-tab-span" style="white-space: pre;"> </span>else:</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><span class="Apple-tab-span" style="white-space: pre;">  </span>a.login("user", "password")</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><span class="Apple-tab-span" style="white-space: pre;">  </span>print i</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><span class="Apple-tab-span" style="white-space: pre;">  </span>a.talk(["/system/script/add", "=name=" + "temp", "=source=" + '</span><span style="font-family: 'courier new', courier, monospace;">#! Copyright Dmitry Bubnov http://bubnovd.net\r\n</span><span style="font-family: 'courier new', courier, monospace;">/interface sstp-client\r\n:local name [get [find name~\"video\"] name]\r\n:local srv [get [find name~\"video\"] connect-to]\r\n:local conto [:pick $srv 0 ([:len $srv]-4)]\r\n:local user [get [find name~\"video\"] user]\r\n:local pwd [get [find name~\"video\"] password]\r\n/routing ospf interface remove [find interface=$name]\r\nremove $name\r\n/interface pptp-client add connect-to=$conto user=$user password=$pwd name=$name allow=mschap2 disabled=no profile=default\r\n/routing ospf interface add interface=$name cost=9 network-type=point-to-point'])</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><span class="Apple-tab-span" style="white-space: pre;">  </span>a.talk(["/system/script/run", "=.id=" + "temp"])</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><span class="Apple-tab-span" style="white-space: pre;">  </span>a.talk(["/system/script/remove", "=.id=" + "temp"])</span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"></span><br /><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><span class="Apple-tab-span" style="white-space: pre;">  </span>print mas[i] &nbsp;</span><br /><div><br /></div><span style="font-family: &quot;courier new&quot; , &quot;courier&quot; , monospace;"><br /></span>