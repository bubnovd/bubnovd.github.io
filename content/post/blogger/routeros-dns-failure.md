+++
title = "RouterOS DNS failure"
date = 2015-03-31T23:32:00Z
updated = 2015-03-31T23:32:27Z
tags = ["routeros", "Mikrotik", "DNS Failure"]
blogimport = true 
[author]
	name = "devi1"
	uri = "https://www.blogger.com/profile/05777499482649623616"
+++

В RouterOS 6.27 добавили несколько фич в DNS резолвер. В некоторых ситуациях адреса просто не резолвятся (DNS failure). Выход простой - в IP-DNS вместо нулей в строках&nbsp;&nbsp;Query Server Timeout и Query Total Timeout прописываем 2. И DNS снова в строю!
