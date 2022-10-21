+++
title = "Коммутаторы D-Link и ERPS"
date = 2016-05-26T06:07:00Z
updated = 2016-05-26T06:07:13Z
tags = ["error", "D-Link", "ERPS"]
blogimport = true 
[author]
	name = "devi1"
	uri = "https://www.blogger.com/profile/05777499482649623616"
+++

Не смог поднять второй ERPS инстанс на D-Link DES-3200. Команда create erps raps_vlan N, введенная второй раз, выдала ошибку:&nbsp;cannot create the r-aps vlan as the maximum number of rings supported has been exceeded.<br /><br />В связи с чем было произведено активное гугление и за неимением результата сего процесса, был отправлен запрос в техподдержку D-Link'a. Ответ привожу здесь:<br /><br />На коммутаторах DES-3200 HW ver.C1 можно создать только одно ERPS кольцо.<br />Серия DGS-3120 rev.B1 поддерживает 4 ERPS кольца, на прошивках R3.00 и выше.<br />Серия DGS-3420 поддерживает 12 ERPS колец, на прошивках R1.5 и выше.<br />Серия DGS-3620 поддерживает 14 ERPS колец, на прошивках R2.30 и выше.
