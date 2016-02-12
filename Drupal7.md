# Введение #

В рамках данной статьи мы займемся вопросом установки [CMS Drupal](http://drupal.org) на нашем роутере. Все необходимое для этого уже имеется в репозитарии этого проекта и доступно через менеджер пакетов opkg.
У нас имеется быстрый web-сервер [lighttpd](http://www.lighttpd.net/), [PHP 5.3.6](http://php.net/) и встроенная поддержка баз данных в виде библиотеки SQLite3.
Приступим...

---

# Установка #

Прежде чем начать установку web-сервера, PHP и Drupal мы должны определится какие именно пакеты нам нужны. А выбор собственно и не велик - т.к. из всех возможных SQL баз данных у нас имеется только SQLite3, использование которой оправдано малым объемом ОЗУ Keentic-а (только 32Мб), то единственная возможная версией которой мы можем воспользоваться - это Drupal7. Только эта версия умеет работать с широким спектром SQL серверов, включая SQLite3, через универсальный PDO драйвер из комплекта PHP 5.3. Все остальные пакеты которые нам будут необходимы диктуются из [минимальных требований Drupal7](http://drupal.org/requirements). Нужно помнить, что объем ОЗУ сильно ограничен, а скорость обмена со swap-файлом на USB носителе не больше 8Мб/с (и это пиковое значение). Поэтому устанавливайте только те пакеты без которых не обойтись, иначе будут большие задержки с отдачей страниц. Список необходимых пакетов таков:
  * Web-сервер:
    * lighttpd _(сам web-сервер)_,
    * lighttpd-mod-access _(модуль контроля доступа к сайту, можно не использовать при работе с анонимным доступом)_,
    * lighttpd-mod-accesslog _(модуль ведения лога обращения к сайту, пригодиться для отыскания проблем или если вас взломают :))_,
    * lighttpd-mod-cgi _(модуль работы с cgi-скриптами, нужен для fastcgi)_,
    * lighttpd-mod-fastcgi _(модуль интерфейса fastcgi через который устанавливается связь с PHP интерпретатором)_,
    * lighttpd-mod-redirect _(модуль перенаправления запросов, нужен для запуска CleanURLs в Drupal и блокировки ботов)_,
    * lighttpd-mod-rewrite _(модуль модификации URL, нужен для запуска CleanURLs в Drupal)_.
  * PHP 5.3 интерпретатор:
    * php5 _(сам интерпретатор)_,
    * php5-cgi _(модуль работы с web-сервер в качестве cgi-скрипта, нужен для fastcgi)_,
    * php5-fastcgi _(модуль работы с web-сервер в через fastcgi)_,
    * php5-mod-dom _(модуль-расширение - HTML DOM, используется Drupal7 core)_,
    * php5-mod-gd _(модуль-расширение - графическая библиотека GD2, используется Drupal7 core)_,
    * php5-mod-hash _(модуль-расширение - построение хеш-сумм и криптография, используется Drupal7 core)_,
    * php5-mod-json _(модуль-расширение - JSON синтаксис, используется Drupal7 core)_,
    * php5-mod-mbstring _(модуль-расширение - UNICODE строки, используется Drupal7 core)_,
    * php5-mod-pdo _(модуль-расширение - PDO универсальный драйвер доступа к SQL базам, используется Drupal7 database API)_,
    * php5-mod-pdo-sqlite _(модуль-расширение - расширение PDO для работы с SQLite/SQLite3, используется Drupal7 database API)_,
    * php5-mod-sqlite3 _(модуль-расширение - прямая работа с библиотекой SQLite3, используется Drupal7 database API)_,
    * php5-mod-session _(модуль-расширение - поддержка HTTP сессий через куки, используется Drupal7 core)_,
    * php5-mod-xml _(модуль-расширение - XML и XML DOM, используется Drupal7 core)_,
    * php5-mod-simplexml _(модуль-расширение - упрощенная работа с XML, используется Drupal7 core, в принципе это аналог модуля php5-mod-xml но Drupal7 использует почему-то и то и другое)_.

Все эти пакеты должны быть установлены в Keenetic-е. Для этого соединяемся с роутером через SSH, заходим под root и выполняем по очереди команду: "**opkg install <имя пакета>**".

Теперь приступим к развертыванию Drupal7. Подготовим для этого место на USB диске Keenetic-а. Для этого на диске нужно создать три каталога:
  * /media/DISK\_A1/net/www - _корневой каталог, где будут расположены скрипты Drupal7_,
  * /media/DISK\_A1/net/db - _каталог, где будет размещена база данных Drupal7_,
  * /media/DISK\_A1/net/log - каталог, где будут расположены логи доступа и ошибок_._

**Обратите внимание**:
> `Все каталоги расположены вне пути /media/DISK_A1/system, который используется менеджером пакетов opkg и OS linux. Это сделано для большей безопасности в случаи взлома web-сервера. Хотя вы вольны расположить их где угодно, например в базовой конфигурации, которую мы конечно будем менять, www каталог предлагается разместить по пути /media/DISK_A1/system.`

Теперь нужно скачать архив дистрибутива Drupal 7.0 по адресу http://drupal.org/project/drupal и распаковать его в каталог /media/DISK\_A1/net/www. Но вызывать Drupal 7 еще рано, прежде нужно настроить конфигурацию lighttpd-сервера, PHP интерпретатора и самого Drupal.


---

# Настройка #

Чтобы все, что мы установили заставить работать - нужно это все сконфигурировать, к чему и приступаем.

**Примечание**:
> `В ходе настройки служб нам нужно будет редактировать их файлы конфигурации. Делать это можно как из под Linux, так и под Windows: единственный нюанс - файлы должны оставаться в Unix-like формате, т.е. код новой строки должен быть LF (так же обозначается \n), а не комбинация CR+LF (\r\n), как принятор для текстовых файлов в Windows`.

## Настройка роутера ##
Прежде чем приступить к настройке lighttpd, php и Drupal нужно изменить кое-что в роутере, а именно: web интерфейс роутера работает на 80-м порту, который пригодится нам для своего сервера. Нужно перевести web интерфейс роутера на другой порт, например на порт 8080. Для этого в разделе "Управление интернет-центром" (Система/Управление) меняем "TCP-порт веб-конфигуратора:" с 80 на 8080. И применяем настройки.

## Настройка lighttpd ##
После установки lighttpd из репозитария его конфигурационный файл находится по пути /media/DISK\_A1/system/etc/lighttpd/lighttpd.conf и настроен он работу со статическим контентом по пути /media/DISK\_A1/system/www, для примера в этом каталоге лежит файл index.html. Кроме того порт для сервера установлен 81-й. Нас это не устраивает, поэтому будем редактировать конфигурационный файл lighttpd.conf. Для этого, найдите в файле lighttpd.conf следующие директивы и замените их значения на указанные. Если директивы закомментированы (первый символ в строке - #), тогда - разкомментируйте их:
  * Подключаем необходимые модули
```
server.modules = ( 
    "mod_rewrite", 
    "mod_redirect",
    "mod_access",
    "mod_cgi",
    "mod_fastcgi",
    "mod_accesslog"
    )
```
  * Меняем порт и тэг сервера:
```
server.port = 80
server.tag = "lighttpd"
```
  * Указываем где сохранять логи работы сервера:
```
server.errorlog = "/media/DISK_A1/net/log/lighttpd/error.log" 
accesslog.filename = "/media/DISK_A1/net/log/lighttpd/access.log"
```
  * Указываем корневую папку со скриптами сервера:
```
server.document-root = "/media/DISK_A1/net/www/"
```
  * Корректируем имена индексных файлов папок (добавляем в конец index.php и default.php):
```
index-file.names = ( "index.html", "default.html", "index.htm", "default.htm", "index.php", "default.php" ) 
```
  * Корректируем правило исключения из набора статических файлов (добавили в конец .inc-файлы, т.е. файлы-вложения, которые используются в php) :
```
static-file.exclude-extensions = ( ".php", ".pl", ".fcgi", ".inc" ) 
```
  * Указываем путь к pid файлу сервера:
```
server.pid-file = "/media/DISK_A1/system/var/run/lighttpd.pid" 
```
  * Подключаем php интерпретатор через интерфейс fastcgi:
```
fastcgi.server = (
    ".php" => (
        "localhost" => (
           "socket" => "/media/DISK_A1/system/tmp/php-fastcgi.socket",
           "bin-path" => "/media/DISK_A1/system/usr/bin/php-fcgi",
           "bin-environment" => ( 
               "PHP_FCGI_CHILDREN" => "",
               "PHP_FCGI_MAX_REQUESTS" => "100" 
            ),
            "max-procs" => 2,
            "idle-timeout" => 20
        )
    )
) 
```
  * Боремся с настырными ботами, которые лазят по инету и пытаются найти и взломать службу phpMyAdmin:
```
$HTTP["url"] =~ "(phpmyadmin)+" { url.access-deny = ("") }
$HTTP["url"] =~ "(/scripts/setup.php)+" { url.access-deny = ("") } 
```
  * Настраиваем службу url.rewrite, чтобы Drupal мог использовать т.н. CleanURL - очень полезно в наших условиях:
```
url.rewrite-if-not-file = (
  "^/system/test/(.*)$" => "/index.php?q=system/test/$1",
  "^/search/node/(.*)$" => "/index.php?q=search/node/$1",
  "^/(.*)\?(.*)$" => "/index.php?q=$1&$2",
  "^/(.*)$" => "/index.php?q=$1"
)
```

Все настройки из перечисленных выше, кроме последней, являются общими и не связаны с CMS Drupal. Они отлично подойдут вам, если вы захотите использовать какую-либо другую CMS или свои скрипты.

На настройке fastcgi нужно остановиться и объяснить ее подробнее. Lighttpd поддерживает динамическое расширение сайтов за счет интерфейса fastcgi. Подробнее про это вы можете прочитать [тут](http://redmine.lighttpd.net/wiki/lighttpd/Docs:ModFastCGI) и [тут](http://redmine.lighttpd.net/wiki/lighttpd/Docs:PerformanceFastCGI). Если говорить кратко, то методика такая: lighttpd устанавливает соединение с указанными в настройках fastcgi сревером (или серверами) через соединение адрес:порт или через сокет. Fastcgi сервер управляет несколькими дочерними процессами, которые должны заниматься обработкой запросов. Как только приходит запрос на lighttpd, который по его настройками относится к какому-либо fastcgi расширению, тогда lighttpd передает этот запрос соответствующему серверу, а тот - свободному дочернему процессу, а если все заняты - запускает еще один.
Наш php интерпретатор умеет работать как fastcgi сервер, но есть одна проблема - он не умеет динамически запускать новые процессы и убивать пустующие. При старте в роли fastcgi сервера он статически создает указанное через переменную окружения **PHP\_FCGI\_CHILDREN** кол-во процессов и постоянно поддерживает их число. По умолчанию это число - 4. Кроме того, мастер-процесс, т.е. тот который работает в роли fastcgi сервер, занимает памяти столько же как и любой дочерний процесс, а это около 11-13 Мб! Теперь представьте нагрузку на роутер с 5-ю запущенными php процессами и каждый по 11Мб и это при том, что ОЗУ всего 32Мб. Вывод один - нам нужно сокращать кол-во запущенных php процессов до рабочего минимума. Идеальное для нас число - 2 процесса, которые могут обрабатывать запросы. При этом от мастер-процесса нужно избавиться тоже - он не нужен в наших условиях, лишняя трата ресурсов. Именно поэтому переменную окружения **PHP\_FCGI\_CHILDREN** для fastcgi-php мы принудительно устанавливаем пустой. Это требование к php fastcgi серверу самому исполнять все запросы, в роли балансировщика нагрузки будет выступать lighttpd. Таких fastcgi серверов мы запускаем два - **max-procs => 2**.

**Примечание**:
> `Почему именно _два_ работающих процесса является подходящим вариантом? В условиях малого объема ОЗУ и медленной работы со swap-файлом желательно чтобы оперативная память была свободна. Поэтому чем меньшим количеством php процессов мы сможем обойтись - тем лучще. Для статических сайтом, где не используется AJAX можно использовать один php процесс. Но, CMS Drupal в своем коде активно использует AJAX и очень часто посылаются два запроса одновременно. Например такая ситуация: вы закачиваете на свой сервер файл или картинку через интерфейс Drupal-а при этом сразу посылается второй запрос, который контролирует когда закончится аплоад. В такой ситуации нужно иметь как минимум два работающих php процесса, чтобы правильно обработать ситуацию. Таким образом два процесса - наиболее подходящий вариант`.

Еще обращаю ваше внимание на то, что параметра **min-procs** для fastcgi интерфейса в lighttpd > 1.4.Х больше нет. А параметр max-procs - это просто кол-во запускаемых fastcgi серверов. Не максимальное число, а - константное. Так же устарел параметр **max-load-per-proc** и он больше не используется.

Теперь обеспечим автозагрузку web-сервера при старте роутера. Для этого найдите в папке /media/DISK\_A1/system/etc/init.d/ файл K50lighttpd и переименуйте в S50lighttpd. Этот файл-скрипт выполняет загрузку и остановку lighttp. В этом скрипте так же есть код, обеспечивающий доступ к серверу с wan (т.е. с Интернета) порта роутера. Правда этот код закомментирован и его нужно сделать рабочим. Откройте файл, найдите и замените строки PORT\_F и iptables на следующие:
```
PORT_F=80

....

iptables -A INPUT -p tcp --dport $PORT_F -j ACCEPT  

....

iptables -D INPUT -p tcp --dport $PORT_F -j ACCEPT 2> /dev/null  
```

## Настройка PHP 5.3 ##
Как и lighttpd, php интерпретатор считывает свои настройки из конфигурационного файла, который расположен по пути - /media/DISK\_A1/system/etc/php.ini. Но кроме этого настройки так же считываются из всех .ini фалов которые расположены в папке /media/DISK\_A1/system/etc/php5, что позволяет нам вносить свои настройки не меняя базового файла php.ini. Такая возможность появилась с добавлением в репозитарий php версии 5.3.6. Если вы загляните в папку /media/DISK\_A1/system/etc/php5, тогда увидите несколько .ini файлов, которые подключают расширения php, установленные нами. Создадим сред них свой файл, пусть это будет myphp.ini следующего содержания:
```
; Language Options
allow_call_time_pass_reference = Off

; Resource Limits
memory_limit = 32M		; Maximum amount of memory a script may consume.
max_execution_time = 120	; Maximum execution time of each script, in seconds.
max_input_time = 120		; Maximum amount of time each script may spend parsing request data.

; Error handling and logging
log_errors = On
log_errors_max_len = 1024

; Paths and Directories
doc_root = "/media/DISK_A1/net/www"


; Module Settings
[Date]
date.timezone = Europe/Kiev

[Session]
session.cookie_path = /
session.bug_compat_42 = Off
session.bug_compat_warn = Off
```

Обратите внимание на параметры **memory\_limit** и **doc\_root**. Они должны быть установлены именно так. В параметр **date.timezone** вы должны вписать свою часовую зону, полный список зон смотрите [на сайте php.net](http://www.php.net/manual/en/timezones.php).
После окончания настройки CMS Drupal, вы можете уменьшить максимальный объем (параметр **memory\_limit**) для php скриптов до 20Мб - этого достаточно для Drupal7 если не ставить дополнительные модули.

## Настройка Drupal ##
Теперь, когда основная работа проделана, можно приступить к настройке и самой CMS. Перегрузите роутер (чтобы стартовал lighttpd и php fastcgi) и зайдите на роутер по адресу _http://192.168.1.1/install.php_. Должен запустится мастер установки Drupal7.
Если все настройки были сделано правильно, как указано выше, вы без труда пройдете установку Drupal от начало до конца.

**Но есть важная особенность**:
> Как оказалось мастер установки Drupal, да и само ядро Drupal иногда неправильно определяет свой базовый URL. В результате процесс установки может не дойти до конца, а если же установка и завершилась - тогда Drupal будет падать в "белый экран смерти". Я, к сожалению, не знаю причин этого. Единственное предположение, что есть некоторые проблемы у PHP - когда он работает через FastCGI под lighttpd, он не может правильно определить базовый URL. Спасибо пользователям форума iXBT ([Sferg](http://forum.ixbt.com/users.cgi?id=info:Sferg), [Fork](http://forum.ixbt.com/users.cgi?id=info:Fork)) которые обратили мое внимание на эту ошибку.
> К счастью из этой ситуации есть выход. Перед началом установки загляните в папку /media/DISK\_A1/net/www/sites/default/, там вы увидите файл _default.settings.php_. Именно этот файл будет использован мастером установки Drupal для создания базовых настроек сайта и результат этой работы будет оформлен в виде файла _settings.php_ в той же папке. Нам нужно самим, до начала работы установщика, переименовать файл _default.settings.php_ в _settings.php_, затем открыть его в редакторе и найти строку
```
 # $base_url = 'http://www.example.com';  // NO trailing slash! 
```
> Эту строку нужно раскоментировать и самим прописать ваш базовый URL под которым сайт виден из Интернета, к примеру для моего сайта эта строка выглядит так:
```
 $base_url = 'http://kyselgov.pp.ua';  // NO trailing slash!
```
> Если вы используете сервис DynDNS, тогда в качестве базового URL должно выступать доменное имя которое Вы для себя зарегистрировали. Еще раз обращаю внимание, что делать это нужно до запуска мастера установки! Но если вы забыли и запустили мастер установки и он у вас благополучно "завис" - ничего страшного. Идите в указанную папку, там уже будет файл _settings.php_, который создал мастер. Не удаляйте его, а просто отредактируйте строку с $base\_url, как указано выше.

Теперь моджно приступить к самой установке, придерживайтесь правил:
  * Процесс установки хорошо описан [на сайте Drupal, 4 шаг процесса инсталляции](http://drupal.org/node/251031).
  * Устанавливайте Drupal только в минимальном виде - Minimal installation profile, чем минимальный профиль отличается от стандартного смотрите [тут](http://drupal.org/node/1127786).
  * В процессе установки путь к файлу базы данных Sqlite3 укажите следующий **/media/DISK\_A1/net/db/.ht.sqlite**.
  * На втором шаге установки мастер может предложит поставить какой-либо еще язык интерфейса. Ни в коем случаи не ставьте его, останьтесь лучше на стандартном English (built-in) language. Дело в том, что перевод интерфейса в Drupal-е сделан [автоматической заменой исполняемого кода в памяти интерпретатора](http://api.drupal.org/api/drupal/includes--bootstrap.inc/function/t/7) строками из базы данных, работает это крайне медленно и очень сильно нагружает базу данных и кушает память - это не наш выбор. Поверьте интерфейс не страдает от того что он на английском языке, тем более что это видно только после логина на сайт. А те немногие фразы которые выводит Drupal на страницы в режиме гостевого входа легко кастомизируются в теме.

Теперь, после установки не начинайте работу с сайтом, а выполните нижеследующие рекомендации:
  * Сразу же включите чистые ссылки - [CleanURLs](http://drupal.org/getting-started/clean-urls). Для этого все уже готово, нужно просто включить их в разделе _Administer > Configuration > Search and metadata > Clean URLs_
  * Теперь деинсталлируйте модуль "**Database Logging**": он ведет полный лог обращений к сайту даже в режиме гостевого входа, а лишняя нагрузка на базу данных нам не нужна.
  * Не устанавливайте модуль "**Path**". Конечно этот модуль очень удобен, он позволяет делать "человеческие" названия ссылкам на страницы. Но его реализация в Drupal 7 очень корявая, на загрузку одной простой страницы модуль "**Path**" делает до 60-ти обращений к базе данных.
  * Заходим в _Administration > Configuration > Performance_ и включаем кеширование страниц: включить _"Cache pages for anonymous users"_ и _"Cache blocks"_, _"Minimum cache lifetime"_ ставим 10 минут, а _"Expiration of cached pages"_ - 1 день, включаем _"Aggregate and compress CSS files"_ и _"Aggregate JScript files"_, _"Compress cached pages"_ - ВЫКЛЮЧАЕМ.
  * Я рекомендую установить модуль [File Cache](http://drupal.org/project/filecache) - очень ускоряет работу с закешированными данными. Для работы этого модуля в файле /media/DISK\_A1/net/www/sites/default/settings.php нужно дописать код:
```
/**
 * turn off HTTP Request Fails message (in admin/reports/status)
 */
$conf['drupal_http_request_fails'] = FALSE;

/**
 * Setings for filecache module
 */ 
$conf['cache_backends'] = array('sites/all/modules/filecache/filecache.inc');
$conf['cache_default_class'] = 'DrupalFileCache';
$conf['filecache_directory'] = '/media/DISK_A1/system/tmp/filecache-' . substr(conf_path(), 6);
```
  * Установите на своем роутере cron (читайте [эту статье](cron_setup.md)). В файл заданий crontab допишите две задачи для нашего сайта:
```
# Drupal cron and VACUUM
10 */1 * * * wget -O - -q -t 1 http://мой_сайт.com/cron.php?cron_key=cron_ИД
* */23 * * * sqlite3 /media/DISK_A1/net/db/.ht.sqlite 'VACUUM' 
```
> первая задача запускает самообслуживание сайта раз в 1час и 10 мин (cron\_ИД вы можете посмотреть в _Administration > Reports > Status report_. Вторая задача "вакуумирует" вашу базу данных - это прямой аналог дефрагментации для файловых систем, выполняется раз в 23 часа.
  * Теперь когда сайт установлен и работает можно снова заглянуть в настройки php и уменьшить объем занимаемой памяти до 20-22Мб - memory\_limit = 22M, так же я рекомендую теперь установить параметры кеширования файлового поиска в php - realpath\_cache\_size и realpath\_cache\_ttl. В результате наш myphp.ini измениться и теперь выглядит так:
```
; Language Options
allow_call_time_pass_reference = Off

; Resource Limits
memory_limit = 22M		; Maximum amount of memory a script may consume.
max_execution_time = 120	; Maximum execution time of each script, in seconds.
max_input_time = 120		; Maximum amount of time each script may spend parsing request data.

; Error handling and logging
log_errors = On
log_errors_max_len = 1024

; Paths and Directories
doc_root = "/media/DISK_A1/net/www"

; Realpath tuning
realpath_cache_size = 128K
realpath_cache_ttl = 12000

; Module Settings
[Date]
date.timezone = Europe/Kiev

[Session]
session.cookie_path = /
session.bug_compat_42 = Off
session.bug_compat_warn = Off
```

В качестве примера перечислю минималистический набор модулей, установленных на моем роутерном сайте.
Стандартные:
  * Block
  * Field
  * Field SQL storage
  * Field UI
  * File
  * Filter
  * Image
  * Menu
  * Node
  * Options
  * System
  * Taxonomy
  * Text
  * User

Дополнительные:
  * File Field Sources
  * IMCE
  * Block for news (мой личный)
  * Colorbox
  * Insert
  * Site Verification
  * File Cache
  * Wysiwyg
  * XML sitemap, XML sitemap engines, XML sitemap node



---

# Завершение #
Наконец-то мы закончили настройки и можем приступить к наполнению сайта содержимым, добавить необходимые модули, недостающее дописать руками... Если вы действительно решили дорабатывать функциональность Drupal-а то лучше это делать в своем модуле, избегайте использования php кода в теле страниц и блоков - не устанавливаейте модуль "**PHP Filter**" - его использование опасно написанием дыр на сайте, так же он отключает систему кеширования страниц Drupal-а.

Пару слов о скорости работы сайта: как уже говорилось - главное чтобы была свободна ОЗУ, чем меньше будет задействоваться своп - тем быстрее будет работать сайт. Максимальную производительность мы будем получить при анонимном/гостевом входе на сайт - скорость отдачи страниц (при условии работы кеша) будет менее 1-2 сек. Когда кеш не работает, а это вариант аутентификационной работы на сайте, задержки могут достигать 8-9 сек. Эти задержки вызваны постоянным парсированием php кода, которого в Drupal-е много. К сожалению попытки прикрутить какую-либо из систем кеширования операционного кода APC, XCache, eAccelerator и т.п. - пока не увенчались успехом :(

P.S.:
За время написания статьи был выпущен новый релиз - Drupal 7.2. Все что было сказано выше - отлично работает и на нем :)

P.P.S.:
(Решение от одного из владельцев кинетика). Если при установке возникают ошибки в файле query.inc, то следует этот файл отредактировать. Заменить

```
protected function removeFieldsInCondition(&$fields, QueryConditionInterface $condition) {
    foreach ($condition->conditions() as $child_condition) {
        if ($child_condition['field'] instanceof QueryConditionInterface) {
          $this->removeFieldsInCondition($fields, $child_condition['field']);
        }
        else {
          unset($fields[$child_condition['field']]);
        }
    }
```
на
```
protected function removeFieldsInCondition(&$fields, QueryConditionInterface $condition) {
    foreach ($condition->conditions() as $child_condition) {
      if (isset($child_condition['field'])) {
        if ($child_condition['field'] instanceof QueryConditionInterface) {
          $this->removeFieldsInCondition($fields, $child_condition['field']);
        }
        else {
          unset($fields[$child_condition['field']]);
        }
      }
    }
  }
```
Файл находиться в папке /media/DISK\_A1/net/www/includes/database/sqlite/query.inc

Лечение взято здесь http://www.drupal.ru/node/86646, http://drupal.org/node/1611828.