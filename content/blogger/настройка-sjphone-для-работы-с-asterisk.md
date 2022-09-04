+++
title = "Настройка SJPhone для работы с Asterisk"
date = 2011-08-11T01:14:00Z
updated = 2013-01-23T01:27:45Z
tags = ["SJPhone", "Asterisk", "IP-телефония", "софтфон"]
blogimport = true 
[author]
	name = "devi1"
	uri = "https://www.blogger.com/profile/05777499482649623616"
+++

Чтобы заставить работать софтфон с вашим сервером IP-телефонии, нужно сначала завести аккаунт на сервере:<br /><br /><a name='more'></a>Для этого жмем Extensions - Add Extension. Выбираем Generic SIP Device. Для тестов пойдут все дефолтные настройки. Нужно лишь вбить UserExtension, DisplayName и secret. Например 100, 100 и 100.<br /><br /><b>Настройка софтфона на примере SJPhone</b><br />Меню - опции - профили -добавить<br />Имя профиля - любое<br /><div class="separator" style="clear: both; text-align: center;"><a href="http://1.bp.blogspot.com/-olLHgVscY1s/TkOOu07ctsI/AAAAAAAAAKE/bkuAb3soEFo/s1600/sj1.png" imageanchor="1" style="margin-left: 1em; margin-right: 1em;"><img border="0" height="223" src="http://1.bp.blogspot.com/-olLHgVscY1s/TkOOu07ctsI/AAAAAAAAAKE/bkuAb3soEFo/s400/sj1.png" width="400" /></a></div>ОК.<br />Переходим на закладку SIP Proxy и указываем адрес нашего сервера<br /><div class="separator" style="clear: both; text-align: center;"><a href="http://2.bp.blogspot.com/-cABWDHuRFv0/TkOPyc1AwVI/AAAAAAAAAKM/Mg125T9rNOM/s1600/sj2.png" imageanchor="1" style="margin-left: 1em; margin-right: 1em;"><img border="0" height="223" src="http://2.bp.blogspot.com/-cABWDHuRFv0/TkOPyc1AwVI/AAAAAAAAAKM/Mg125T9rNOM/s400/sj2.png" width="400" /></a></div><br /><div class="separator" style="clear: both; text-align: center;"></div>ОК. Вылезет табличка с запросом имени и пароля. В неё надо вбить то, что указывали при регистрации Extension'a.
