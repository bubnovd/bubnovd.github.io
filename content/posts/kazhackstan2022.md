---
title: KazHackStan 2022
date: "2022-09-19T04:22:41Z"
author: bubnovd
authorTwitter: bubnovdnet
cover: "/img/khs22/logo.jpg"
description: Посетил конференцию по ИБ в Алмате. Кратко, о том, что увидел, услышал и успел записать дилетант из докладов о безопасности и безопасной разработке
tags:
- kazhackstan
- security
- conference
keywords:
- security
- burp
- nginx
- devsecops
- appsec
- redteam
showFullContent: false
readingTime: true
hideComments: false
---

Посетил конференцию по ИБ в Алмате. Кратко, о том, что увидел, услышал и успел записать дилетант из докладов о безопасности и безопасной разработке

## Вызовы безопасной гибридной облачной инфраструктуры
Спикер: Эльдар kyprizel Заитов (Yandex)
- для IAM можно использовать Azure AD и SCIM
- для аудита нужен реестр сервис аккаунтов [cloudquery](https://www.cloudquery.io/)

## Как сделать SAMM оценку процессов appsec, и как с этим работать в будущем
Спикер: Ольга Свиридова
- SAMM: Security Assurance Maturity Model от OWASP. Есть аналог от MS. Модель обеспечения зрелости безопасности. Позволяет оценить текущий уровень безопасности и спланировать его улучшение

## Как мы запускали Национальную BugBounty платформу
Казахстан - одна из немногих стран (работающая программа есть в Сингапуре. И, кажется, это всё), запустивших государственную программу багбаунти. Багбаунти - это официальная выплата исследователям (читай - хакерам), нашедшим уязвимости. В случае  с национальным bugbounty выплата полагается за уязвимости на государственных/муниципальных ресурсах. 
Один из принятых и оплаченных багов - пароль admin на систему управления водоснабжением столицы государства. Подробнее можно почитать на [хабре](https://habr.com/ru/post/666072/)

## DevSecOps для самых маленьких
Спикер: Иван Кондратьев

Классный доклад для таких дилетантов как я. Показали ктипичный DevSecOps пайплайн и популярные инструменты:
### Container Security:
- Gitlab Container Scanning (CE)
- Trivy
- JFrog XRay
- Clair
- Harbor
- Quay
- Docker hub

### DAST:
- Gitlalb DAST (EE)
- OWASP ZAP
- Tenable.io
- Acunetix
- Synopsys DAST
- Fortify Webinspect

### Защита инфры:
- Контроль доступа: RBAC
- Сетевая безопасность: Security Groups / Network Policy 
- Управление секретами: Vault, Lockbox
- Авторизация SSO: Keycloak / Dex + LDAP
- Policy as Code: Open Policy Agent / Gatekeeper 
- Аудит и мониторинг: Falco, Kyverno, SIEM


## Специфика расследования инцидентов в контейнерах
Спикер: Дмитрий Евдокимов (Luntry)

Контейнеры - маложивущая сущность и заниматься расследованием инцидентов в них - задача неблагодарная. Но не стоит забывать, что контейнер это чаще всего обычный процесс в Linux, а значит можно использзовать старые инструменты или написать над ними удобную обертку


## Защита мобильных приложений от реверса и темперинга
Спикер: Роман Евтеев

Всегда восхищался реверсерами и бинарщиками. Ничего не понятно,. но очень интересно =)
Показали один из методов обфускации кода (сейчас может быть много неточностей, т.к. я в этом ничего не понимаю): внутри .apk лежит самодельная виртаульная машина на python, которая расшифровывает и запускает лежащие рядом зашифрованные исполняемые файлы.
Кажется, это самый хардкорный доклад на конференции. Не перестаю восхищаться ребятами, которые способны так глубоко копать


## Защищаемся от автоматизации спамеров и хакеров
Спикер: Антон Лопаницын (Bo0oM)

Антон рассказал про тюнинг nginx для борьбы с ботами

- Модуль для проверки сертификатов [NGINX JA3 ssl fingerprint module](https://github.com/phuslu/nginx-ssl-fingerprint)
- [Nginxconfig.io](Nginxconfig.io)
- В nginx кроме tar.gz есть ещё сжатие Brotli. Это алгоритм от гугла
- Nginx reuseport
- [The Illustrated TLS 1.2 Connection](https://tls12.xargs.org/). Там же есть [TLS 1.3](https://tls13.xargs.org/)  и [другие](https://xargs.org/)

## Open Source Projects Responsibility Awareness (by Open BLD Example)
Спикер: Евгений Гончаров

Зачем делать OpenSource и кому это нужно. Доклад от казахского Столлмана. Респект Евгению за все его проекты. Кстати, почитайте про [Open BLD](https://lab.sys-adm.in/ru) и даже можете потестировать или поставить домой - сервис вполне рабочий. Сам им пользуюсь

## Впервые в Казахстане! Только у нас! Или анализ UEFI-трояна
Спикер: Олег Биль

Ещё один хардкорный доклад. Правда, без технических деталей, поэтому хардкорным он бы мог быть, если б не NDA. Спикер рассказал о буткитах - вирусах, которые живут внутри UEFI. И которые крайне сложно обнаружить и, тем более, избавиться от них. Переустановка ОС не освобождает от вируса, замена HDD тоже не дает результата. Зловред наинает работу ещё до загрузки ОС, после загрузки внедряет себя в svchost. Файла в системе, естественно нет.
Ещё один классный подход в том, что вирус не лезет за пределы локалки - он работает с интернетом через прокси - другой вирус, работающий на соседнем компе. Поэтому даже по сетевым соединениям отследить его непросто.
Захотелось послушать другие доклады этого спикера. Видимо, они должны быть интересными

## Инфраструктура для проведения тестирований на проникновение в формате Red Teaming
Спикер: Вадим Шелест

Второй доклад для _самых маленьких_
Сначала ссылки из презентации:
- https://byt3bl33d3r.substack.com/
- https://opsec101.org/
- [Cmon, stop using scope.txt and bugs.txt / Дмитрий Частухин / VolgaCTF 2021](https://www.youtube.com/watch?v=aazM16elhtI)
- [Золотой век Red Teaming С2 фреймворков — Вадим Шелест](https://www.youtube.com/watch?v=pZ3lBX3s7-U)

![redteam](/img/khs22/redteaminfra.jpg)

- [Nebula](https://byt3bl33d3r.substack.com/p/taking-the-pain-out-of-c2-infrastructure-3c4) - Overlay Network by Slack
- [WarHorse](https://github.com/warhorse/warhorse) - IaC for RedTeam Infra
- Hive. Project Management for RedTeam 
- GoPhish
- Mailgun
- Evilginx и GoPhish имеют защиту от скрипткидди
- Pwndrop
- CC: Covenant, PoshC2, Sliver, Silenttrinity, CobaltStrike



## Получение доступа через компрометированные учетные данные – Как взломщики находят и эксплуатируют секреты
Спикер: Mackenzie 

Немного ужасов про захардкоженые секреты в git и docker
- [Отчет THE STATE OF SECRETS SPRAWL 2022](https://www.gitguardian.com/state-of-secrets-sprawl-report-2022)
- инструмент для проверки сикретов в инфре: [ggshield](https://github.com/GitGuardian/ggshield)



В одном из докладов услышал интересную мысль: CTF - синтетическая штука, слабо связанная с реальностью. Спикер рассказал о тренировке апсеков прямо на своём продукте. Кто-то придумывает CTF внутри сети и команда решает его.

Ещё тулы и ссылки:
- [DefectDojo](https://github.com/DefectDojo/django-DefectDojo) is a DevSecOps and vulnerability management tool. 
- [Black duck](https://www.synopsys.com/software-integrity/security-testing/software-composition-analysis.html) - software composition analysis 
- Safebase.io
- Censys.io
- Autorize - плагин к берпу, turbo intruder, param miner
- Arjun, mitmproxy, http mock
