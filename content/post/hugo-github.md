---
title: Деплой статических сайтов в GitHub Pages
date: "2023-01-08T07:36:06Z"
author: bubnovd
authorTwitter: bubnovdnet
image: "/img/hugo-github/logo.jpg"
description: Деплой статических сайтов в GitHub Pages с помощью GitHub Actions
tags:
- static site generator
- hugo
- github actions
- github pages
keywords:
- static site generator
- hugo
- github actions
- github pages
showFullContent: false
readingTime: true
hideComments: false
spendTime: 3h
---

- разобрать что за Static Sites, какие есть
- пример сайта на hugo
- как тестировать сайт локально с помощью Docker
- что такое github pages
- что такое github actions
- деплой статик сайта в github pages с помощью github actions
- привязывание кастомного домена к этому сайту
- книга
- импорт из других движков

_Обложка: Млечный Путь с хижины Рацека 24.09.2022_

## Немного истории
В начале развития веба _(web 1.0 - уточнить годы и версию веба)_ сайты были статическими. На странице был текст и картинки. Для любого пользователя сайт выглядел одинаково и взаимодействие с польователем было не нужно. Тенхлогии равивались, пользователям хотелось ваимодействовать с сайтами - оставлять комментарии, обсуждать новости. Сайтов сатло больше и их администрирование нужно доверить нетехническим людям - упростить добавление новостей и новых страниц. Для этого понадобились так называемые админки. Для реализзации этого требовались обработчики действий пользователей. Сначала это был Perl, потом PHP и другие. 
По прошествии какого-то рвемени стало ясно, что не всем нужны динамические сайты. Нарпимер, сайт-визитка компании не нуждается ни в комментариях, ни в форме заказа. Так же как  персонаьный блог не требует упрощенной админки. Наступила новая эра статических сайтов.

## Static Sites
Статические сайты не требуют использования дополнительных ьехнологий: никаких баз данных, движков обработки - только веб сервер. Если раньше админ сайта вручную редактировал html код - требовались знания HTML, CSS. В новом мире появились генераторы статических сайтов. От админа требуется написать текст (обычно в формате Markdown), прикрепить картинки, выбрать тему сайта и запустить генератор. На выходе получится  набор html, css, js файлов для работы сайта. Достаточно просто положить эти файлы в папку с веб-сервером и получим полноценный сайт. 
Существует несколько распространенных генераторов статических сайтов. _внений вид, плагины, функциональность_

- __Hugo__: Написан на Go. В качестве шаблонов использует Go Template. Легко настраивается, очень быстро генерирует сайт. Не требует зависимостей - всего один бинарник
- __Jekyll__: Написан на Ruby. Шаблоны Liquid. Один из первых генераторов статических сайтов с огромным комьюнити. Хорошо оптимизирован для SEO
- __Gatsby__: Написан на JavaScript, шаблоны React. Для работы требует GraphQL. Позволяет делать PWA - Progressive Web Application. Огромное количество плагинов
-  __[Lektor](https://xakep.ru/2017/08/08/lektor-get-started/), Next, Zola, Eleventy, Pelican, ...__

Все движки имеют огромное количество тем для разных применений

## Делаем сайт на Hugo
В большинстве случаев для создания сайта нужно склоинровать репозиторий с генератором и выполнить несколько команд:
- создать структуру каталогов
- создать новый пост
- подготовить сайт  публикации
В итоге получим структуру директорий, которую нужно положить в директорию, откуда веб-сервер читает контент. Да, вот так просто.

Работать с hugo будем из Docker. Инструкция по его установке и настройке есть на [официальном сайте](https://docs.docker.com/). Если читателю не хочется ставить Docker, то предлагаю [разбраться]](https://gohugo.io/installation/) с установкой Hugo - это совсем не сложно.

Заранее создадаим репозиторий с сайтом на GitHub и склонируем его (`git clone https://github.com/bubnovd/hugosite.git`). В первых шагах он нам не нужен, но пригодится дальше при публикации сайта

Создадаим новый сайт с именем hugostite. Имя сайта должно совпадать с именем репозитория.
export HUGO_DOCKER_TAG=0.107.0-ext
`docker run -it --rm -v $(pwd):/src -p 1313:1313 klakegg/hugo:$HUGO_DOCKER_TAG new site hugosite --force`
force в данном случае используем потому что директория уже есть - мы скачали её из гитхаба, а переписывать не хотим, т.к. там конфиг гита

Заходим в директорию hugosite. Добавим тему therminal
`git submodule add -f https://github.com/panr/hugo-theme-terminal.git themes/terminal`
И пропишем в конфиге, что будем использовать именно её
`echo "theme = 'terminal'" >> config.toml`

Можно запускать сайт. Сейчас он пустой
`docker run -it --rm -v $(pwd):/src -p 1313:1313  klakegg/hugo:0.107.0-ext serve -D --bind 0.0.0.0`
[ext тут нужен потому что не работает без него](https://gohugo.io/troubleshooting/faq/#i-get--this-feature-is-not-available-in-your-current-hugo-version)
![run-theme](/img/hugo-github/run-theme.png)

Создадим новый пост `docker run -it --rm -v $(pwd):/src -p 1313:1313  klakegg/hugo:0.107.0-ext new posts/my-first-post.md`
Генератор создал файл  content/posts/my-first-post.md с содержимым
```
+++
title = "My First Post"
date = "2023-01-08T11:05:20Z"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["", ""]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++
```

Тут можно изменить название поста, проставить теги, выбрать цвет и другие параметры. Ниже последних `+++` можно писать текст в формате Markdown
C таким содержимым полуилось как на картинке 
```
+++
title = "Hello Ninjas!"
date = "2023-01-08T11:05:20Z"
author = "Mikrotik Ninja"
authorTwitter = "bubnovdnet" #do not include @
cover = ""
tags = ["hugo", "demo"]
keywords = ["hugo", "demo"]
description = "Tiny post for hugo demo post"
showFullContent = false
readingTime = true
hideComments = false
color = "green" #color from the theme settings
+++

# Hello Ninjas!
_This is first post_
- first
- second
- third
```

![first-post](/img/hugo-github/first-post.png)
Запускаем, `docker run -it --rm -v $(pwd):/src -p 1313:1313  klakegg/hugo:0.107.0-ext serve -D --bind 0.0.0.0` проверяем, что работает localhost:1313

В [документации к Mrkdown](https://www.markdownguide.org/basic-syntax/) можно почитать на что способен этот формат и офрмлять посты красиво. Так же сам Hugo и темы к нему имеют огромное количество настроек, с помощью которых можно изменить свой сайт как угодно. В этом посте не будем заниматься внешним видом - об этом уже много написано, а нам ещё нужно опубликовать сайт и автоматизировать его деплой.

Теперь зальем наш получившийся репозиторий на GitHub. 
`git commit -m hugosite && git push`

## Публикация сайта на GitHub
Hugo умеет работать на любом вебсервере, а так же имеет средства для деплоя на популярные облачные сервисы:
- 21YunBox
- AWS Amplify
- Azure Static Web Apps
- Netlify
- Render
- Firebase
- GitHub
- GitLab
- KeyCDN
- Cloudflare Pages
В официальной документации Hugo описаны все перечисленные виды деплоя.

Проект [GitHub Pages](https://pages.github.com/) может сделать сайтом любой публичный репозиторий. Для этого достаточно в настройках репозитория активировать Pages и указать ветку, которая будет использоваться в качестве сайта. 
![github-pages](/img/hugo-github/github-pages.png)
Для деплоя нашего сайта понадобится ещё одна фича GitHub'a

### GitHub Actions
Как платформа для хранения исходного кода, GitHub просто обязан был внедрить возможность CI/CD. Эта функциональность обеспечивается [GitHub Actions](https://github.com/features/actions). В корне репозитория создается файл `.github/workflows/gh-pages.yml`. В нем описываются действия, которые будут происходить при определенных событиях. Например, при пуше в ветку develop необходимо сначала собрать проект, а затем запустить тесты. А при мердже в main опубликовать проект. По сути Actions запускает контейнер с каким-то приложением и передает ему параметры запуска. 

В нашем случае `.github/workflows/gh-pages.yml`. выглядит так:
```
name: github pages

on:
  push:
    branches:
      - main  # Set a branch that will trigger a deployment
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v3
        with:
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          extended: true

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        if: github.ref == 'refs/heads/main'
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

Воркфлоу работает на Ubuntu 22.04 и состоит из трех шагов:
- Первый шаг исспоьзует action checkout версии 3. Он скачивает наш репозиторий вместе с сабмодулями, что указано в соответствующем поле
- Шаг Setup Hugo устанавилвает в контейнер с Ubuntu пакет Hugo последней версии `hugo-version: 'latest'`. Правила хорошего тона требуеют указывать конкретные версии пакетов, чтобы не сломать продукт, разработанные для старой версии. Дело в том, что тег `latest` диктует использовать последнюю версию Hugo, в то время, как наш сайт тестировался локально с версией 0.107.0-ext. Когда выйдет новая версию Hugo, то Actions будет использовать её. Есть вероятность, что какие-то старые фичи перестант поддерживаться в новой версии и наш сайт сломается. Так что ВСЕГДА указывате конкретные теги, вместо `latest`
- В шаге Build запускается hugo в режиме генерации сайта. В нем из наших исходников в папку public собираются html, css и js файлы, которые мы используем при октрытии сайта в браузере
- Шаг Deploy указывает использовать папку public, полученную в предыдущем шаге, в качестве содержимого веб-сервера. Тут же указывается `GITHUB_TOKEN`, требуемый для публикации новой ветки с сайтом

`GITHUB_TOKEN` откуда берется
указать бранч в пэеджес

В репозитории на вклладке Actions можно увидеть весь процесс сборки и деплоя сайта. Каждый шаг можно развернуть и посмотреть его логи
![actions](/img/hugo-github/actions.png)
