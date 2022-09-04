+++
title = "Проверка пропускной способности канала на Mikrotik"
date = 2017-12-21T08:52:00Z
updated = 2017-12-21T08:52:00Z
tags = ["bandwidth", "routeros", "Mikrotik", "script"]
blogimport = true 
[author]
	name = "devi1"
	uri = "https://www.blogger.com/profile/05777499482649623616"
+++

Скрипт для Mikrotik RouterOS, тестирующий пропускную способность канала и записывающий результат в файл. Перед запуском создать файл isp-quality.txt и указать IP своего btest сервера в ipbandswtestserver<br /><br /><a href="https://github.com/devi1/RouterOS-scripts/tree/master/bandwidth%20test">https://github.com/devi1/RouterOS-scripts/tree/master/bandwidth%20test</a><br /><br /><br />:local txAvg 0<br />:local rxAvg 0<br />:local ipbandswtestserver your.btest.server.ip<br />:local btuser btest<br />:local btpwd btest<br />:local ts [/system clock get time]<br />:local ContentsFile [/file get isp-quality.txt contents]<br />:local ds [/system clock get date]<br /><br />:set ts ([:pick $ts 0 2].[:pick $ts 3 5].[:pick $ts 6 8])<br />:set ds ([:pick $ds 7 11].[:pick $ds 0 3].[:pick $ds 4 6])<br /><br />tool bandwidth-test protocol=tcp direction=transmit address=$ipbandswtestserver duration=5s do={<br />:set txAvg ($"tx-total-average" / 1048576 );<br />}<br /><br />tool bandwidth-test protocol=tcp direction=receive address=$ipbandswtestserver duration=5s do={<br />:set rxAvg ($"rx-total-average" / 1048576 );<br />}<br /><br />/file set isp-quality.txt contents="$ContentsFile\n$ds-$ts tx: $txAvg Mbps - rx: $rxAvg Mbps"<br /><br /><br />
