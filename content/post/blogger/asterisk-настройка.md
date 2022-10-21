+++
title = "Asterisk настройка"
date = 2010-11-11T20:02:00Z
updated = 2010-11-11T20:02:23Z
blogimport = true 
[author]
	name = "devi1"
	uri = "https://www.blogger.com/profile/05777499482649623616"
+++

Для удобной настройки IP-телефонии потребуется установить веб-интерфейс - через него всё делать гораздо удобнее (если вы - гуру линукс, то, наверное, вам будет проще редактировать конфигурационные файлы вручную).<br />Итак, установка FreePBX:<br /><br /><ol><li><i>cd /usr/src</i></li><li>Выкачиваем установочные файлы <i>wget http://mirror.freepbx.org/freepbx-2.8.0.tar.gz</i>&nbsp;</li><li>Распаковываем<i> tar -xzvf freepbx-2.8.0.tar.gz</i></li><li><i>cd freepbx-2.8.0</i></li><li>Внимательно читаем INSTALL (<i>cat INSTALL</i>). В этом файле описано всё, что нужно для установки, в частности пакеты: libxml2, libxml2-devel,&nbsp;libtiff,&nbsp;libtiff-devel,&nbsp;lame,&nbsp;httpd (or Apache2),&nbsp;mysql (or mysql-client),&nbsp;mysql-devel (or libmysqlclient10-dev), mysql-server,&nbsp;php (or php4) ,&nbsp;php4-pear,&nbsp;php-mysql,&nbsp;php-gd,&nbsp;openssl,&nbsp;openssl-devel (or libssl-dev),&nbsp;kernel-devel (or linux-source),&nbsp;&nbsp;&nbsp; &nbsp; perl,&nbsp;perl-CPAN,&nbsp;bison,&nbsp;ncurses-devel (or libncurses5-dev),&nbsp;audiofile-devel (or libaudiofile-devel),&nbsp;curl,&nbsp;&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;sox. Проверяем, есть ли в нашей системе эти пакеты (<i>rpm -q %название_пакета% </i>(естественно без знака %)). Если пакет есть система покажет нам его версию. Если какого-то пакета нет - устанавливаем его. Я делаю это через yum (<i>yum install %название_пакета%</i>).</li></ol>
