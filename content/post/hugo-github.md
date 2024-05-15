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
spendTime: 7h
---


_Обложка: Окрестности Иссык-Куля 01.10.2022_

>В статье рассмотрим генераторы статических сайтов, создадим сайт на одном из них, опубликуем его в GitHub Pages и настроим автодеплой в Gitub Actions

## Немного истории
В начале развития веба сайты были статическими. На странице был текст и картинки. Для любого пользователя сайт выглядел одинаково и взаимодействие с польователем было не нужно. Тенхологии равивались, пользователям хотелось ваимодействовать с сайтами - оставлять комментарии, обсуждать новости. Сайтов становилось больше, они требовали взаимодествия с пользователями и их наполнение нужно доверить нетехническим людям - упростить добавление новостей и новых страниц - нужны простые админки. Для реализации этого требовались обработчики действий пользователей. Сначала это был Perl, потом PHP и другие. 
По прошествии какого-то времени стало ясно, что не всем нужны динамические сайты. Например, сайт-визитка компании не нуждается ни в комментариях, ни в форме заказа. Так же как  персональный блог не требует упрощенной админки. Наступила новая эра статических сайтов.

## Static Sites
Статические сайты не требуют использования дополнительных технологий: никаких баз данных, движков обработки - только веб сервер. Раньше админ сайта вручную редактировал html код - требовались знания HTML, CSS. В новом мире появились генераторы статических сайтов. От админа требуется написать текст (обычно в формате Markdown), прикрепить картинки, выбрать тему сайта и запустить генератор. Он обработает тексты постов и создаст красивый сайт в соответствии со своими шаблонами. На выходе получится  набор html, css, js файлов для работы сайта. Достаточно просто положить эти файлы в папку с веб-сервером и можно пользоваться полноценным сайтом. 
Существует несколько распространенных генераторов статических сайтов:
- __Hugo__: Написан на Go. В качестве шаблонов использует Go Template. Легко настраивается, очень быстро генерирует сайт. Не требует зависимостей - всего один бинарник
- __Jekyll__: Написан на Ruby. Шаблоны Liquid. Один из первых генераторов статических сайтов с огромным комьюнити. Хорошо оптимизирован для SEO
- __Gatsby__: Написан на JavaScript, шаблоны React. Для работы требует GraphQL. Позволяет делать PWA - Progressive Web Application. Огромное количество плагинов
-  __[Lektor](https://xakep.ru/2017/08/08/lektor-get-started/), Next, Zola, Eleventy, Pelican, ...__

Все движки имеют огромное количество тем для разных применений. Для своего блога я выбрал генератор Hugo - он показался мне достаточно удобным, с нужным функционалом, а так же использует Go Template, с которым было интересно разобраться, так как такие темплейты использует большинство современного ПО для клауд сервисов.

> Книга про генераторы статических сайтов [Static Site Generators. Modern Tools for Static Website Development](https://www.oreilly.com/library/view/static-site-generators/9781492048558/)

## Делаем сайт на Hugo
В большинстве случаев для создания сайта нужно склоинровать репозиторий с генератором и выполнить несколько команд:
- создать структуру каталогов
- создать новый пост
- подготовить сайт к публикации
В итоге получим структуру директорий, которую нужно положить в директорию, откуда веб-сервер читает контент. Да, вот так просто.

Работать с hugo будем из Docker. Инструкция по его установке и настройке есть на [официальном сайте](https://docs.docker.com/). Если читателю не хочется ставить Docker, то предлагаю [разобраться]](https://gohugo.io/installation/) с установкой Hugo - это совсем не сложно.

Заранее создадаим репозиторий с сайтом на GitHub и склонируем его `git clone https://github.com/bubnovd/hugosite.git`. В первых шагах он нам не нужен, но пригодится дальше при публикации сайта

Зафиксируем используемую версию Hugo, чтобы в будущем при обновлении не менять версии в командах. Вместо этого будем указывать переменную `HUGO_DOCKER_TAG`:

`export HUGO_DOCKER_TAG=0.107.0-ext`

Создадим новый сайт с именем hugostite. Имя сайта должно совпадать с именем репозитория:

`docker run -it --rm -v $(pwd):/src -p 1313:1313 klakegg/hugo:$HUGO_DOCKER_TAG new site hugosite --force`

force в данном случае используем чтобы Docker не ругался ну существующую директорию - мы скачали её из гитхаба, а переписывать не хотим, т.к. там конфиг гита.

Заходим в директорию hugosite:

`cd hugosite`

Добавим тему therminal:

`git submodule add -f https://github.com/panr/hugo-theme-terminal.git themes/terminal`

И пропишем в конфиге, что будем использовать именно её:

`echo "theme = 'terminal'" >> config.toml`

Теперь можно запускать сайт. Сейчас он пустой

`docker run -it --rm -v $(pwd):/src -p 1313:1313  klakegg/hugo:$HUGO_DOCKER_TAG serve -D --bind 0.0.0.0`

[ext тут нужен потому что эта тема не работает с базовым Hugo](https://gohugo.io/troubleshooting/faq/#i-get--this-feature-is-not-available-in-your-current-hugo-version). Для других тем нужно отдельно проверить заработает ли Hugo с базовым образом (без -ext).
![run-theme](/img/hugo-github/run-theme.png)

Создадим новый пост `docker run -it --rm -v $(pwd):/src -p 1313:1313  klakegg/hugo:$HUGO_DOCKER_TAG new posts/my-first-post.md`

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

Запускаем, `docker run -it --rm -v $(pwd):/src -p 1313:1313  klakegg/hugo:$HUGO_DOCKER_TAG serve -D --bind 0.0.0.0` проверяем, что работает - открываем браузер по адресу `localhost:1313`.

В [документации к Markdown](https://www.markdownguide.org/basic-syntax/) можно почитать на что способен этот формат и оформлять посты красиво. Так же сам Hugo и темы к нему имеют огромное количество настроек, с помощью которых можно изменить свой сайт как угодно. В этом посте не будем заниматься внешним видом - об этом уже много написано, а нам ещё нужно опубликовать сайт и автоматизировать его деплой.

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

Проект [GitHub Pages](https://pages.github.com/) может сделать сайтом любой публичный репозиторий. Для этого достаточно в настройках репозитория активировать Pages и указать ветку, которая будет использоваться в качестве сайта. Укажем её позже, после первого деплоя, когда появится нужная нам ветка.

Сайт в Pages будет иметь адрес username.github.io или username.github.io/reponame, что позволяет держать несколько сайтов на одном GitHub аккаунте.  Кроме того сайт можно опубликовать на своем домене. Для этого нужно будет пройти [верификацию домена](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/about-custom-domains-and-github-pages)

Теперь для обновления сайта нужно локально генерировать файлы с помощью Hugo и заливать их в репозиторий, но мы автоматизируем процесс с помощью ещё одной фичи GitHub'a.

## GitHub Actions
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
          hugo-version: '0.107.0'  # Our Hugo version
          extended: true           # extended Hugo 

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
- Первый шаг использует action checkout версии 3. Он скачивает наш репозиторий вместе с сабмодулями, что указано в соответствующем поле
- Шаг Setup Hugo устанавливает в контейнер с Ubuntu пакет Hugo указанной версии `hugo-version: '0.107.0'`. Правила хорошего тона требуеют указывать конкретные версии пакетов, чтобы не сломать продукты, разработанные для старой версии. По умолчанию тут стоит `latest`. Дело в том, что тег `latest` диктует использовать последнюю версию Hugo, в то время, как наш сайт тестировался локально с версией 0.107.0-ext. Когда выйдет новая версию Hugo, то Actions будет использовать её. Есть вероятность, что какие-то старые фичи перестанут поддерживаться в новой версии и наш сайт сломается. Так что ВСЕГДА указывате конкретные теги, вместо `latest`
- В шаге Build запускается hugo в режиме генерации сайта из исходников, скачанных в первом шаге. В нем из наших исходников в папку public собираются html, css и js файлы, которые мы используем при открытии сайта в браузере
- Шаг Deploy указывает использовать папку public, полученную в предыдущем шаге, в качестве содержимого веб-сервера

В репозитории на вкладке Actions можно увидеть весь процесс сборки и деплоя сайта. Каждый шаг можно развернуть и посмотреть его логи
![actions](/img/hugo-github/actions.png)

Для корректной работы GitHub Pages нужно указать ветку, которая будет публиковаться. В нашем случае это ветка [gh-pages](https://gohugo.io/hosting-and-deployment/hosting-on-github/#branches-for-github-actions).
![github-pages](/img/hugo-github/github-pages.png)

Во время деплоя, скорее всего, получим ошибку `Error: Action failed with "The process '/usr/bin/git' failed with exit code 128"`
![error](/img/hugo-github/error.png)

Это значит, что GitHub Action не хватает прав для пуша в репозиторий. Нужно выдать эти права в настройках репозитория `Actions - General - Workflow Permissions: Read and Write Permissions`

![wf-permissions](/img/hugo-github/wf-permissions.png)

Помимо сборки и деплоя сайта в воркфлоу можно добавить проверку ошибок, сжатие картинок и всё, что пожелает админ. Думаю, читатель сам найдет и другие способы применения этого продукта. Кроме GitHub Actions можно использовать GitLab CI, Travis CI, Circle CI, Jenkins и другие CI/CD системы.

## Импорт постов из других источников
Если у вас уже есть блог на какой-либо платформе, то в новый блог можно легко импортировать записи с этой платформы. Есть огромное количество скриптов для переноса содержимого из других платформ. Если же у вас не очень распространенная платформа, для такой скрипт можно написать самому. Вот список работающих импортеров с [официального сайта Hugo](https://gohugo.io/tools/migrations/):
- Jekyll
- Ghost
- Octopress
- DokuWiki
- WordPress
- Medium
- Tumblr
- Drupal
- Joomla
- Blogger
- Contentful
- BlogML

Для переноса статей с blogger я пользовался скриптом [blogger-to-hugo](https://pypi.org/project/blogger-to-hugo/). Посты переехали в отдельную папку blogger. Некорректно переехали переносы строк, но это поправимо. Большим преимществом является то, что в таких импортированных статьях можно проставить алиасы адресов для постов. Например, в старом блоге пост лежал по адресу https://www.bubnovd.net/2015/03/mikrotik-routeros.html, а в hugo он окажется по адресу https://bubnovd.net/blogger/самый-полный-мануал-по-резервированию-интернета-на-mikrotik-routeros и если у кого-то был сохранен линк на старый блог, то этот линк просто не откроется. В hugo в заголовке поста можно указать алиас для адреса и пост будет лежать уже по двум путям - старому и новому. Так мы не сломаем линки на наши посты со сторонних ресурсов.


Подходы к работе с сайтами сильно изменились со времен первого веба и теперь создать красивый современный сайт под силу каждому. А попутно ещё можно разобраться с современными технологиями, чтобы не отставать от быстро мчащегося паровоза под названием Информационные Технологии