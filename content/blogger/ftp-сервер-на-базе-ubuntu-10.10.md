+++
title = "FTP-сервер на базе Ubuntu 10.10"
date = 2011-01-16T23:46:00Z
updated = 2011-01-17T00:06:24Z
draft = true
blogimport = true 
[author]
	name = "devi1"
	uri = "https://www.blogger.com/profile/05777499482649623616"
+++

Будем ставить ProFTPD.<br /><br /><ol><li>Устанавливаем пакет:&nbsp;<span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace;">sudo apt-get install proftpd. </span><span class="Apple-style-span" style="font-family: inherit;">Отвечаем на появившийся вопрос: "самостоятельно".</span></li><li><span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace;">sudo nano /etc/shells </span><span class="Apple-style-span" style="font-family: inherit;">Добавляем строчку </span><span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace;">/bin/false</span></li><li><span class="Apple-style-span" style="font-family: inherit;">Создаем в каталоге /home папку FTP-shared(имя можно любое).</span><span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace;"> sudo mkdir /home/FTP-shared</span></li><li><span class="Apple-style-span" style="font-family: inherit;">Создаём юзера</span><span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace;"> sudo useradd userftp -p password -d /home/FTP-shared -s /bin/false </span><span class="Apple-style-span" style="font-family: inherit;">Вместо password нужно вписать что-то своё, состоящее <b>не только</b> из цифр.</span></li><li><span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace;">В FTP-shared создаём две вложенные папки с любыми именами sudo mkdir /home/FTP-shared/public</span></li></ol><span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace;">sudo mkdir /home/FTP-shared/upload</span><div><ol><li><span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace;"><br /></span></li></ol><div><span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace;"><br /></span></div></div>
