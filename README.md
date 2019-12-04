# Virtual_Disk_For_Linux_Kernel_5
Виртуальный диск (блочное устройство) для Linux Kernel 5

Всем привет, в момент изучения чего-то, хочется придумать с одной стороны несложную задачу, а с другой стороны интересную и что-бы кто-то мог получить какую-то пользу от твоего проекта.

С эти и сталкнулся я при изучении ядра Линукс и Винды, да поставил перед собой цель изучить эти две системы на низком уровне.

Да все начинают изучение с создания простого драйвера символьного устройства, это своебразный "Хеллоу ворлд" для разработчиков ядра, далее можно усложнять добавив мультиплексирование, блокировки и т.д.

Но такое выкладывать стыдно, даже на своем форуме, а-то мне уже сказали, что не круто выкладывать хеллоуворлды, в целом согласен с сообществом.

Поэтому решил поковырять драйвера блочных устройств, стало интересен тот факт, что в новом ядре Линуксе 5.0, перешли на blk-mq и тем самым даже официальный пример блочного устройство перестал собираться...

Вообще если интересно поковырять драйверы для Линукса, классные публикации:

https://www.ibm.com/developerworks/ru/library/l-linux_kernel_01

Более новый материалл, про блочные устройства с Хабра:

https://habr.com/ru/company/veeam/blog/446148/

Так-вот решил поставить себе задачу для треннировки:

1)Написать драйвер виртуального диска, который можно отформатировать в любую ФС.

2)Размер, число виртуальных дисков должно передоваться в модуль, через параметры.

3)Драйвер должен иметь дефолтные параметры, взял такие: 

После загрузки модуля будет создаваться четыре виртуальный диска, размером четыре мб каждый, с именем "xd".

4)Драйвер должен собираться и работать в ядре 5, вообще оттестировал на Ubuntu 19, там как-раз 5-е ядро, на Ubuntu 16.04 несобралось, т.к. разные структуры и функции работы с очередями.

Вот-что получилось:

1. Создание числа дисков и их размера, в зависимости от параметров:
# insmod block_mod_s.ko size=4 ndevices=4 

ГДЕ:

size - Размер накопителей, в мегабайтах.

ndevices - Число накопителей.

Есть ещё дополнительные параметры загрузки модуля:

major и hardsect_size

2.Диски можно отвформатировать в любую ФС, и подмонтировать стандартной утилитой mount.

3. Работает с ядром 5+.

С другими ядрами не работает.

Вообще немного теории в чём-тут отличия и что не сделано, т.к. проект тестовый:

Традиционно подсистема для работы с драйверами блочных устройств в Linux предоставляла два варианта передачи запроса драйверу: один с очередью, и второй — без очереди.

В первом случае используется очередь для запросов к устройству, общая для всех процессов. С одной стороны, такой подход позволяет использовать различные планировщики, которые в общей очереди могут переставлять запросы местами, замедляя одни и ускоряя другие, объединять несколько запросов к соседним блокам в единый запрос для снижения издержек. С другой стороны, при ускорении обработки запросов со стороны блочного устройства эта очередь становится узким местом: одним на всю операционную систему.

Во втором случае запрос отправляется напрямую в драйвер устройства. Конечно, это даёт возможность миновать общую очередь. Но в тоже время, возникает необходимость создавать свою очередь и планировать запросы внутри драйвера устройства. Такой вариант подходит для виртуальных устройств, которые обрабатывают запросы и перенаправляют транзитом на реальные устройства.

В последние несколько лет скорость обработки запросов блочными устройствами выросла, в связи с чем требования к операционной системе, как и к интерфейсу передачи данных, изменились.

В 2013 году был разработан третий вариант — multiqueue block layer (blk-mq).
Запрос сначала попадает в программную очередь. Количество этих очередей равно количеству ядер процессора. После прохождения программной очереди запрос попадает в очередь отправки. Количество очередей отправки уже зависит от драйвера устройства, который может поддерживать от 1 до 2048 очередей. Так как работа планировщика осуществляется на уровне программных очередей, то запрос из любой программной очереди может попасть в любую очередь отправки, предоставляемую драйвером.

Т.к. проект тестовый, то создается одна очередь для устройства, которая связывается с "Желзкой".

Также для более низких ядер, проект несоберется, если нужно можно собрать тоже и под 3-4 ядра.

Тестирование модуля:

#Загрузка модуля (С дефолтными параметрами):
# sudo insmod virtual_disk.ko

#Просмотр списка устройств:
# sudo ls -l /dev/x*

#Вывод геометрии устройства:
# sudo hdparm -g /dev/xda

#Форматирование устройства:
# sudo mkfs.ext2 /dev/xda
# sudo fsck /dev/xda

#Монтирование устройства в папку media:
# sudo mount -t ext2 /dev/xda /media

Ну далее уже можно размещать файлы и т.д.

#Вывод диагностических сообщений модуля:

# sudo cat /proc/kmsg
