+++
title = "Memory"
date = "2022-09-11T11:52:28Z"
author = ""
authorTwitter = "bubnovdnet"
cover = "/img/vm/Linux_logo.jpg"
tags = ["OS", "Linux", "Memory"]
keywords = ["OS", "Linux", "Memory", "OSTEP"]
description = "Виртуальная память в Linux"
showFullContent = false
readingTime = true
hideComments = false
+++

Большинство из нас пользуется 64-разрядными системами. В таких системах процессы работают с адресным пространством размером 2^64 байт (16 эксабайт или 16 млн ТБ). Многие из читателей имеют столько памяти в системе? Как получается, что процесс видит 16 ЭБ памяти, когда в системе её намного меньше? Как получается, что относительно небольшая физическая память так интенсивно используется программами и не позволяет читать чужие данные? Как распределяется между процессами физическая память?

На эти вопросы отвечает книга `Operating Systems. Three easy pieces`. До её прочтения я слабо представлял себе работу ОС. А теперь представляю ещё хуже, зато более системно =) Вы ведь знакомы с этим чувством, когда начинаешь изучать что-то новое и открываешь новую пропасть в своих знаниях, которую придется заполнять ещё долгие годы?

Во время чтения книги появилась идея написать серию постов о работе ОС. Хочу начать с виртуальной памяти, так как мне её работа показалась очень элегантной и эффективной.

## Итак, виртуальная память в Linux. Часть 0

Чем хороша виртуализация памяти: прозрачность, эффективность, изоляция
- Прозрачность (transparency ). Virtual memory реализована операционной системой абсолютно незаметно для процессов. Программы не знают что там происходит с памятью. Они думают, что владеют всей памятью в системе и не заботятся об изоляции от дургих проессов

- Эффективность (efficiency). Программисту не нужно думать о том, как и где хранить переменные, потому что виртуальное адресное пространство огромно. Жизнь становится проще, если не нужно задумываться о работе с низкоуровненвыми абстракциями. ОС делает виртуализацию максимально эффективной по времени и размеру. Благодаря работе с железом ОС делает этот процесс ещё лучше (MMU, TLB)

- Изоляция. ОС обеспечивает изоляцию адресных пространств. Один процесс не может обратиться к памяти другого. Спцеиально или из-за бага

---

- Процесс Петя имеет счет в банке PM и приложение VM, которое показывает количество денег

## Виртуальная память в Linux. Часть 1

Давным давно системы были однозадачными и вся память была доступна одному процессу. Не надо было беспокоиться о приватности данных и безопасности. Потом наступила эра многозадачности: компьютером стали пользоваться одновоременно несколько человек. Каждый запускает свои программы. Ранние реализации просто отдавали всю память процессу, а когда приходил другой процесс - содержимое памяти первого сбрасывалось на диск и в неё загружались данные нового процесса.

![13.1-early-days.png](/img/vm/13.1-early-days.png)

Очевидно, что запись и чтение с диска - операция затратная и куча времени бесполезно расходовалась на это. Нужен был новый, более эффективный процесс.

Этот процесс называется Sharing. Это использование одной физической памяти одновременно несколькими процессами. У каждого процесса есть своё адресное пространство
![13.2-sharing-memory-3-processes](/img/vm/13.2-sharing-memory-3-processes.png)

Адресное пространство (Address Space) - память, которую видит процесс. В 32 разрядных системах это 2^32 байт (4 ГБ), в 64 разрядных 2^64 Б (16 ЭБ). Очевидно, что физическая память не имеет отношения к объёму адресного пространства. Не у каждого из нас ведь есть 16 ЭБ RAM. Это всего лишь абстракция, предоставляемая операционной системой

Сам Address Space делится на три части:
- Code. Код процесса
- Heap. Куча. Начинается сразу после кода и растет вниз (от stack к 2^32 в 32-разрядных системах)
- Stack. Стек. Начинается с конца адресного простарнства и растет вверх (с 2^32 в 32-разрядных системах до конца кучи)

![13.3-address-space](/img/vm/13.3-address-space.png)

Как видно на картинке 13.3 процесс считает, что адресация его памяти начинается с ноля. КАЖДЫЙ процесс так считает. И все они одновременно работают в физической памяти

Как ОС создаёт эту иллюзию приватного огромного адресного пространства, начинающего с нуля для каждого процесса, на единой, не всегда большой физической памяти?

---


## Виртуальная память в Linux. Часть 2. Стек, куча и код

В прошлый раз мы увидели, что адресное пространство процесса делится на три части: код, куча и стек. Рассмотрим их подробнее.

- Code. Первая часть адресного пространства. Начинается с 0. Тут содержится код исполняемой программы. Эта часть памяти может быть доступна другим процессам. Ведь таким образом мы можем сэкономить память при запуске нескольких идентичных процессов

- Heap. Начинается сразу после кода и растет вниз. Система заранее не знает сколько памяти нужно выделить по heap. Да и это было бы неэффективно. Поэтому под heap выделяется какая-то инициализационнная часть памяти. Когда процессу понадобится больше памяти для heap, он запросит её у ОС через системный вызов malloc(). Heap эксплицитен. Память для него нужно выделять в программе явно и чистить после себя. Освобождается память системным вызовом free(). В heap хранятся связанные списки, хэш таблицы, деревья и другие структуры данных 

- Stack. Начинается с конца адресного пространства и растет вверх. Стек управляется компилятором имплицитно. Поэтому он называется автоматической памятью. Когда функция выполнилась, она вернула значение и компилятор деаллоцировал память. В стеке хранятся локальные переменные, параметры функций и возвращаемые адреса

![13.3-address-space](/img/vm/13.3-address-space.png)

Понятно, почему стек и хип разнесли в разные стороны адресного пространства. Так они могут расти, не мешая друг другу

Конечно, адресное пространство - всего лишь абстракция, предоставляемая процессам со стороны ОС. Вся память, которую видит процесс - абстракция. Поэтому она и называется виртуальной памятью (Virtual Memory). Адреса виртуальной памяти указывают на память физическую. Преобразованием виртуальных адресов в физические занимается ОС с помощью различных аппаратных ухищрений

---

Это и есть виртуализайия
!изляция



---
Дальше моё сочинительство


Каждый процесс считает, что ему доступно 2^32 (4096 МБ) или 2^64 (16 млн ТБ) байт памяти. Но как это возможно, если в системе памяти значительно меньше?
Наверняка многие слышали о понятии "виртуальная ппамять". Эта фича ОС и позволяет обманывать процессы, предлагая им для испоьзования столько памяти, сколкьо в систее просто нет.

Попробуем разобраться как это работает.
Адресное простарнство - вся память, доступная проессу. Для простоты будем рассматривать 8-битную систему. Адресное пространство в ней будет равно 2^8 = 256 байт. 
Процесс использует для работы три элемента (три счета): 
!Описать каждый
- код. Статичен. Всегда занимает одинаковое кол-во памяти. Можно шарить между процесаами?
- стэк
- куча

Заранее никто не может знать сколько памяти нужно стеку или хипу. Поэтому для них выделяется какая-то инициализационная часть памяти с возможностью расширения. Чтобы не мешать друг другу они занимают адресное пространство с противополоных сторон и растут в разные стороны. Стек растет вверх (от большего к меньшему: 256 -> 100), хип растет вниз (от меньшего к большему: 10 -> 50)
```
  -------------- 0
  |    Code    |
  -------------- 10
  |    Heap    |
  |     |      |
  |     v      |
  -------------- 50
  |   (free)   |
  |     ...    |
  -------------- 100
  |     ^      |
  |     |      |
  |   Stack    |
  -------------- 256
```

Получается, что в 64-разрядной системе каждый процесс считает, что ему доступно 16 экзабаййт памяти. Сколько раз вы видели компьютер с таким количеством оперативы? Вот и в нашей восьмибитной системе нет 256 байт памяти, а есть только 100







# Interlude: Memory API

вставить здесь спойлер
- почитать про malloc()
- почитать про sizeof()
- почитать про free()
- это не сисколы, а лайбраари колы
- почиттаь про brk(), sbrk - change the location of the program’s break: the location of the end of the heap. Они не могут использоваться напрямую. Они ывзываются через malloc и free
- почитать про mmap()

# Mechanism: Address Translation

Как мы видели ранее каждый процесс считает, что его адресное пространство начинается с 0 и имеет доступ к огромному объёму памяти (2^64 Б) ![13.3-address-space](/img/vm/13.3-address-space.png). Хотя в физической памяти адресное пространство процесса может располагаться как угодно 
![13.2-sharing-memory-3-processes](/img/vm/13.2-sharing-memory-3-processes.png)

Трансляция адресов: преобразование виртуального адреса в физический. Обеспечивается ОС вкупе с аппаратной поддержкой

_Тут приводятся упрощения, что адресное пространство размещено в памяти последовательно (смежно), адресное пространство меньше физической памяти и все адресные пространства одного размера_

Для понимания работы трансляции (релокации), обратимся к ранним решениям этой задачи, примененным ещё в 1950-х. Эта техника называется base and bound (не смог корректно перевести на русский. Что-то вроде "база и граница") или динамическая релокация.

Каждое ядро CPU имеет два регистра: base (базовый) и bound (граничный).
- В base регистре лежит смещение. `Физический адрес = виртуальный адрес + base`
- bound регистр указывает на границу физической памяти процесса. Если адрес, к которому обращается процесс выходит за границу, указанную в bound регистре, вызывается исключение и процесс завершашется

Пример:
```
Address Space Size 4 KB
Loaded in phys address 16 KB
• Virtual Address 0 → Physical Address 16 KB
• VA 1 KB → PA 17 KB
• VA 3000 → PA 19384
• VA 4400 → Fault (out of bounds)
```

Регистры - часть CPU и работу по релокации делает CPU. Часть процессора, которая занимается работой с памятью называется Memory Management Unit (MMU). Чем сложнее эта работа, тем более сложным будет MMU

Проблемы:
- При создании процесса ОС должна найти место в физ памяти для размещения там адресного пространства процесса. То есть нужно иметь список свободного места (free list)
- После завершения процесса нужно освободить его место в физ памяти  и записать обратно в free list
- У ЦПУ только одна пара бэйз-баунд регистров. При конекст свитчинге нужно сохранить куда-то бэйз и баунд текущего процесса и считать откуда-то эти регистры нового процесса. Procss Control Block (PCB)
- Доступ к регистрам привилигрованный. Только кернел режим ОС может ими управлять. Если бы это мог делать обычный процесс, он бы мог перезаписать значения регистров и читать/писать чужую память
- Мы смогли создать независимую память для каждого процесса и изолировать её. Но т.к. аллоцируем всё адресное пространство, то аллоцируемое, но неиспользуемое место между хипом и стеком простаивает впустую, как видно на рис. 15.2. Это называется внутренней фрагментацией
![15.2-base-bound](/img/vm/15.2-base-bound.png)

В следующей части попробуем решить эти проблемы

---

# Segmetation

Для предовтарщения внутренней фрагментации можно вместо одной пары регистров использовать три. По паре на каждый сегмент адресного пространства: код, стек, хип
![16.2-segments-in-phys-mem.png](/img/vm/16.2-segments-in-phys-mem.png)

```
  -------------- 0 KB
  |  Operating |
  |   System   |
  -------------- 16 KB
  | not in use |
  |            |
  --------------
  |     ^      |
  |     |      |
  |   Stack    |
  --------------
  | not in use |
  |            |
  -------------- 32 KB
  |    Code    |
  --------------
  |    Heap    |
  |     |      |
  |     V      |
  -------------- 
  |            |
  |            |
  |    not     |
  |     in     |
  |    use     |
  |            |
  |            |
  |            |
  -------------- 64 KB
```


Получаем такую структуру в ММУ

Segment | Base | Size
--------|------|-------
Code    | 32K   | 2K
Heap    | 34K   | 2K
Stack   | 28K   | 2K

Я не буду пересказывать тут как система обращается к разным сегментам памяти. Интересующиеся могут почитать об этом [в книге](https://t.me/mreadninja/118)

<!---
Обратимся к адресу 100 в этой таблице. Добавляем сдвиг 100 к Base. Получаем 32К + 100 = 32768 + 100 = 32868. Этот адрес входит в предел 32К + 2К. Это КОД

![an-address-space.png](/img/vm/an-address-space.png)

Попробуем обратиться к адресу 4200, который должен быть в хипе. 34К + 4200 = 34816 + 4200 = 39016. 39016 > 36864. Полученный адрес не входит в сегмента хипа.  Но так как это новый сегмент (не код). Нужно получить сдвиг (offset). Так как хип начинается с 4 КБ (в соответствии с ![an-address-space.png]) и нам нужен адрес 4200, то офсет будет 4200-4096=104. Получаем 34816 + 104 = 34920
Получли Segmentation Fault - обрщение к памяти за пределелами нашего адресного пространства

Но как обратиться к стеку? Ведь он растет в обратном направлении? Сначала посчитаем наше место в адресном пространстве. При обращении к виртуальному адрсу 15K. Он мапится на физический адрес 27 КБ. 15 КБ = 11 1100 0000 0000 . Старшие два бита определяют сегмент, значит на смещение остается 3 КБ (РАСПИСАТЬ ТУТ КАК ЭТО ПОЛУЧАЕТСЯ). Чтобы получить корректный отрицательный сдвиг, надо вычесть из него максимальный размер сегмента (откуда он берется? Посмотреть в домашке) 3 КБ. Получаем  3 - 4 = -1. Добавляем полученный результат к base 28 - 1 = 27 КБ
-->

Иногда требуется шарить некоторые сегменты между адрес спейсами. Например шаринг сегмента с кодом. Для этого используется protection bit. Выставив его в read-only мы позволим читать из сегмента другим процессам

Итак, сегментация позволила нам не расходовать впустую огромные куски памяти с минимальным оверхедом на трансляцию
![16.3-external-fragmentation.png](/img/vm/16.3-external-fragmentation.png)

Проблема заключается в том, что сегментация все еще недостаточно гибкая, чтобы поддерживать наше полностью обобщенное, разреженное адресное пространство. Например, если у нас есть большой, но редко используемый хип, расположенный в одном логическом сегменте, весь хип все равно должен находиться в памяти, чтобы к нему можно было получить доступ. Другими словами, наша модель использования адресного пространства не совсем соответствует тому, как работает сегментация. Таким образом, нам необходимо найти новые решения


# Free-Space Management

# Paging

Наша цель - виртуализация памяти. Использовать одну физическую память разными процессами. Безопасно и эффективно. Сегментация решает эти задачи, но имеет ряд проблем: управление свободной памятью сложно, т.к. фрагментация и сегментация не настолько гибки, как хотелось бы. Получаем куски неиспользуемой памяти.

На помощь приходит пейджинг. Вместо деления адресного пространства на три логических сегмента (каждый - динамического размера) делим адресное пространство на **фиксированные** блоки - страницы

![18.1-simple-address-space.png](/img/vm/18.1-simple-address-space.png)

Физическая память тоже делится на страницы. В них будут размещаться страницы адресного пространства. Страницы физической памяти называются page frame

![18.2-paging-physical.png](/img/vm/18.2-paging-physical.png)

Каждая виртуальная страница (VPN - Virtual Page Number) указывает на физический фрейм (PFN - Physical Frame Number). Для их сопоставления используется Page Table - таблица страниц. Для каждого процесса она своя и находится по адресу /proc/PID/pagemap. Но открывать этот файл бесполезно - каждая страница это 64-битное (для 64-битных систем) значение. Для страниц размером 4 КБ это 2^52 страниц по 8 байт каждая. Это очень много данных и открытие такого файла займет годы. В /proc/PID/maps можно увидеть все используемые страницы.

Сам виртуальный адрес состоит из двух частей: VPN и offset (сдвиг). VPN указывает на страницу, а сдвиг - на смещение байт в этой странице. В трансляции адресов участвует толкьо VPN. Сдвиг остается прежним.

![18.3-address-translation.png](/img/vm/18.3-address-translation.png)

На самом деле в Page Table содержатся не только данные о сопоставлении виртуальных и физических страниц. Например:
- `valid bit` - указывает на корректность трансляции - что такая страница существует
- `protection bits` - уровевнь доступа (read/write/executable)
- `present bit` - есть ли страница в физической памяти или в свопе (swap)
- `dirty bit` - изменялась ли страница после её помещения в своп
- ...

Пейджинг гибкий, упрощает управление свободным пространством. С ним нет внешней фрагментации, поскольку используются страницы одинакового размера. Освободившийся физический фрейм легко занимается новой страницей того же размера. Однако пейджинг всё ещё дорогая операция и не всегда эффективная (чтобы занять 1 бит в памяти, требуется выделить целых 4 КБ на страницу)


# Translation Lookaside Buffer (TLB)

Пейджинг - тяжелая операция. Как это часто бывает с научными и техническими инновациями, в первые годы его считали неэффективным и бесполезным. Понадобилось больше десяти лет, чтобы оптимизировать его и использовать в промышленных системах.

В оптимизации этого процесса помогла, как часто бывает, аппаратная составляющая. Page Table в отдельную структуру внутри CPU - Translation Lookaside Buffer. Это ещё одна функция уже известного нам MMU.

Кроме этого, для оптимизации работы Page Table используются разные её организации: линейная, уменьшенная, инвертированная




readelf

numactl
size