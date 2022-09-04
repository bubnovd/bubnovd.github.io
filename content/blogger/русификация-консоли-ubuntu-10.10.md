+++
title = "Русификация консоли Ubuntu 10.10"
date = 2011-01-16T23:23:00Z
updated = 2011-01-16T23:23:45Z
tags = ["Ubuntu 10.10", "консоль", "русификация"]
blogimport = true 
[author]
	name = "devi1"
	uri = "https://www.blogger.com/profile/05777499482649623616"
+++

При подсоединении к Ububntu 10.10 с помощью PuTTY язык нормальный, русские буквы видно. А вот если работать с локальной консолью (tty1-tty6), то при выводе русских букв (например из apt-get) получаются кракозябры.<br />Чтобы избавиться от этого, необходимо сделать следующее:<br /><br /><ol><li><span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace;">sudo apt-get install console-cyrillic</span></li><li><span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace;">sudo dpkg-reconfigure console-cyrillic</span></li></ol><span class="Apple-style-span" style="font-family: inherit;">И в появившейся менюшке выбрать то, что надо (главное выбрать Unicode). И перезагрузить систему.</span>
