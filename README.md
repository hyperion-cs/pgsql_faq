# :elephant: PostgreSQL – FAQ

Ответы на часто задаваемые вопросы начинающих DBA / FAQ for DBA beginners.

## Введение

:exclamation: В первую очередь, мы должны уметь думать **самостоятельно**.
В особенности, это касается проблем, которые у вас возникают
— сначала тщательно и усердно попытайтесь [нагуглить](https://www.google.com/) решение, и только в случае неудачи задавайте вопросы (_как это делать правильно — см. ниже_).<br>
Уважайте время и труд других участников сообщества, общайтесь исключительно в вежливой манере.
Это первостепенные и необходимые условия для того, чтобы приумножать свои навыки и расти как профессионал.

## Оглавление / Table of Contents

<!-- Тэг <br> необходим для корректной работы вложенных списков. -->

1. [Сообщество](#1-community)
2. [Вопросы](#2-questions)<br>
    2.1. [Не могу подсоединиться к PostgreSQL. Что делать?](#2.1-connection-problem)<br>
    2.2. [Получаю ошибку что имя таблицы/колонки/etc неверное, хотя это не так. В чем дело?](#2.2-name-does-not-exist)<br>
    2.3. [Как публиковать SQL запросы, определения функций, выводы команд вспомогательных утилит и прочую текстовую информацию при запросе помощи у сообщества?](#2.3-data-publication)<br>
    2.4. [У меня медленно работает SQL запрос и я хочу попросить помощи у сообщества. Какую информацию мне необходимо предоставить?](#2.4-slow-query)<br>
    2.5. [У меня не получается написать SQL запрос/etc и/или я получаю ошибки. Какую информацию мне необходимо предоставить для получения помощи от сообщества?](#2.5-invalid-query)<br>
    2.6. [В каком регистре писать команды/функции/etc в PostgreSQL?](#2.6-register-codestyle)<br>
3. [Книги/курсы](#3-learning)
4. [Полезные ссылки](#4-links)
5.  [Переводы / Translations](translations)<br>
    5.1. [English](translations/ENG.md)


## <a name='1-community' /> [1.](#1-community) Сообщество :family_man_woman_girl_boy:

У PostgreSQL прекрасное вежливое сообщество, которое обладает заметным свойством толерантности к начинающим участникам.
Среди основных точек входа стоит отметить следующее:

1. [Чат русскоязычного сообщества PostgreSQL в Telegram](https://t.me/pgsql);
2. [PGDay.ru](https://pgday.ru/) - Ежегодная конференция по PostgreSQL (Санкт-Петербург);
3. [PGConf.ru](https://pgconf.ru/) - Ежегодная конференция по PostgreSQL (Москва).


## <a name='2-questions' /> [2.](#2-questions) Вопросы :interrobang:

### <a name='2.1-connection-problem' /> [2.1.](#2.1-connection-problem) Не могу подсоединиться к PostgreSQL. Что делать?

Пожалуй, это наиболее частый вопрос у начинающих пользователей PostgreSQL, ежедневно звучащий в информационной среде сообщества.
Как правило, они получают ошибки, содержащие ключевые слова `Connection refused` или `Connection ... failed`. Например:
```
error: connection to server at "localhost", port 5432 failed: Connection refused 
```
или (в случае попытки подключения через [unix domain socket](https://en.wikipedia.org/wiki/Unix_domain_socket)):
```
error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: No such file or directory
    Is the server running locally and accepting connections on that socket?
```

Следует провести диагностику проблемы поэтапно (_в настоящее время инструкция применительна только для Unix-based ОС_):

#### 2.1.1. Запущен ли основной процесс PostgreSQL?

Посмотреть, запущен ли основной процесс PostgreSQL, можно выполнив след. команду на сервере:

```bash
ps -ef | grep "postgresql.*config_file"
```

В случае, если PostgreSQL запущен, в выводе команды вы должны увидеть нечто вроде:
```bash
<PG_PATH> -D <DATA_DIR> -c config_file=<CONFIG_PATH>
```

Где `<PG_PATH>` — путь до исполняемого файла PostgreSQL,
    `<DATA_DIR>` — путь до директории с данными кластера,
    `<CONFIG_PATH>` — пусть до конфигурационного файла `postgresql.conf` (потребуется для дальнейшей диагностики).

В противном случае, проблема найдена: PostgreSQL не запущен.

#### 2.1.2. Верен ли TCP-порт, открываемый сервером PostgreSQL?

Теперь необходимо убедиться, что PostgreSQL слушает нужный вам TCP-порт (если, конечно, вы не собираетесь работать только через Unix-сокеты).
Откройте конфигурационный файл `postgresql.conf` (путь до него мы определили этапом выше) и найдите параметр `port`.
Там должно быть значение необходимого вам порта, который по умолчанию равен `5432`.
В противном случае, исправьте номер порта на нужное вам значение. После применения изменений необходимо произвести рестарт PostgreSQL.

_\* В то же время, можно скорректировать значения порта непосредственно на клиенте (том самом, где возникла ошибка соединения)
— всё зависит от конкретной цели, которую вы преследуете._

После рестарта PostgreSQL можно удостовериться, что нужный вам порт действительно прослушивается.
Для этого можно выполнить след. команду на сервере (для порта `5432`):
```bash
ss -ln | grep ":5432"
```

В случае успеха, в выводе команды вы должны наблюдать одну или несколько строк с операцией LISTEN, которые показывают, что PostgreSQL действительно прослушивает порт.
В противном случае (учитывая, что шаг выше показал, что основной процесс СУБД запущен),
вероятно, вы неверно указали значение `port` либо настроена работа только через Unix-сокеты — как это исправить см. ниже.

#### 2.1.3. Верны ли адреса TCP/IP, по которым PostgreSQL принимает подключения?

Параметром в файле `postgresql.conf`, отвечающим за то, через какие [сетевые интерфейсы](https://ru.wikipedia.org/wiki/%D0%A1%D0%B5%D1%82%D0%B5%D0%B2%D0%BE%D0%B9_%D0%B8%D0%BD%D1%82%D0%B5%D1%80%D1%84%D0%B5%D0%B9%D1%81)
PostgreSQL будет принимать соединения, является `listen_addresses`.

Если вы хотите подключиться к PostgreSQL локально (т.е. с того же сервера), подойдут значения
`127.0.0.1` (для подключения по IPv4), `::1` (для подключения по IPv6) или `localhost` (в современных ОС данное доменное имя, как правило, транслируется в `::1`. Подробнее см. [тут](https://ru.wikipedia.org/wiki/Localhost)).
Стоит отметить, что в параметре `listen_addresses`, как следует из его названия, можно указать несколько значений через запятую.
Учтите, что старые клиентские приложения, в подавляющем большинстве случаев, работают по IPv4.

Если вы хотите подключиться к PostgreSQL удаленно (т.е. с другого сервера), то необходимо принимать подключения с соотв. внешних интерфейсов.
С помощью значения `0.0.0.0` можно принимать подключения со всех адресов IPv4, а `::` — все адреса IPv6.
В то же время, если указать значение `*`, то PostgreSQL будет принимать подключения со всех имеющихся сетевых интерфейсов.

Подробнее о подключениях и аутентификации к PostgreSQL см. [тут](https://postgrespro.ru/docs/postgresql/15/runtime-config-connection#RUNTIME-CONFIG-CONNECTION-SETTINGS).
Напомним, что после применения изменений в файл `postgresql.conf` необходимо произвести рестарт PostgreSQL.

:exclamation: В случае, если вы указали PostgreSQL прослушивать внешние интерфейсы
(т.е., что-то, отличное от значений `127.0.0.1`, `::1` или `localhost` параметра `listen_addresses`),
то ваш сервер может быть доступне извне (т.е. из Интернета) и потенциально находится под угрозой.
Чтобы избежать негативных последствий, необходимо корректно настроить ваш [firewall](https://ru.wikipedia.org/wiki/%D0%9C%D0%B5%D0%B6%D1%81%D0%B5%D1%82%D0%B5%D0%B2%D0%BE%D0%B9_%D1%8D%D0%BA%D1%80%D0%B0%D0%BD) и файл `pg_hba.conf`. Подробнее о них см. ниже.

#### 2.1.4. Верна ли конфигурация аутентификации клиентов PostgreSQL?

Несмотря на то, что параметр `listen_addresses` в файле `postgresql.conf` настроен верно, PostgreSQL всё ещё может отвергать соединения.
Причиной может быть то, что в конфигурационном файле `pg_hba.conf` нет соответствующего разрешения.

Подробнее о файле `pg_hba.conf` см. [тут](https://postgrespro.ru/docs/postgresql/15/auth-pg-hba-conf) (включая примеры настройки).

#### 2.1.5. Верна ли конфигурация firewall? 

Даже если PostgreSQL настроен верно (с точки зрения подключения клиентов), вы всё ещё можете иметь неудачные попытки подключения.
В этом случае вам необходимо обратить внимание на правильность настройки firewall (он может быть как на уровне ОС, так и на уровне ваших сетевых аппаратных/виртуальных устройств, напр., роутера).
Конкретные шаги выходят за рамки данного FAQ.

### <a name='2.2-name-does-not-exist' /> [2.2.](#2.2-name-does-not-exist) Получаю ошибку что имя таблицы/колонки/etc неверное, хотя это не так. В чем дело?

В подавляющем большинстве случаев, дело в том, что имя таблицы/колонки (равно как и любого другого символьного идентификатора) содержит символы в верхнем регистре (т.е. заглавные буквы).
В то же время, по умолчанию, PostgreSQL преобразовывает указанные в запросе/команде идентификаторы в нижний регистр.
Чтобы избежать ошибки, в запросе/команде необходимо заключить идентификатор в двойные кавычки (в этом случае вышеописанное преобразование будет отключено).

Пример воспроизведения и решения проблемы:

```sql
SELECT * FROM TableName; -- error: relation "tablename" does not exist
SELECT * FROM "TableName"; -- OK
```

Обратите внимание, что идентификатор `TableName` без двойных кавычек был преобразован в `tablename`, что вызвало ошибку.

Для того, чтобы даже гипотетически избежать таких проблем, не рекомендуется использование символов верхнего регистра в идентификаторах.
Однако, это ни в коем случае не является обязательным правилом.

### <a name='2.3-data-publication' /> [2.3.](#2.3-data-publication) Как публиковать SQL запросы, определения функций, выводы команд вспомогательных утилит и прочую текстовую информацию при запросе помощи у сообщества?

Прежде всего, **не используйте** скриншоты для того, чтобы показать SQL запросы (и/или их результаты), определения функций,
выводы команд вспомогательных утилит (таких, как [psql](https://postgrespro.ru/docs/postgresql/15/app-psql) и др.) и прочую текстовую информацию.
Заметно эффективнее будет публикация оных, напр., на [gist](https://gist.github.com/) или [pastebin.com](https://pastebin.com/).
Если вы задаете вопрос в tg-чате и кол-во содержимого невелико, его допускается опубликовать в режиме [форматирования Monospace](https://telegra.ph/markdown-07-07) прямо в чат.
Как правило, этот вопрос тесно связан с п. [2.4](#2.4-slow-query) и п. [2.5](#2.5-invalid-query).

Многие участники сообщества принципиально не рассматривают скриншоты (и на это есть рациональные причины), поэтому постарайтесь оформить свой текст правильно.

### <a name='2.4-slow-query' /> [2.4.](#2.4-slow-query) У меня медленно работает SQL запрос и я хочу попросить помощи у сообщества. Какую информацию мне необходимо предоставить?

Чтобы получить качественную помощь по оптимизации запроса и не уничтожить желание у сообщества помочь вам, необходимо немного постараться и собрать некоторые данные.<br>
Как в своё время [метко подметил](https://t.me/pgsql/303899) один уважаемый пользователь tg-чата, минимальная информация для получения помощи следующая:

1. Версия PostgreSQL на сервере, где запускается запрос;
2. Непосредственно SQL запрос;
3. Вывод метакоманды \d из утилиты [psql](https://postgrespro.ru/docs/postgresql/15/app-psql) для каждой используемой в запросе таблицы;
4. `EXPLAIN (ANALYZE, BUFFERS)` для запроса (подробнее об EXPLAIN см. [тут](https://postgrespro.ru/docs/postgresql/15/sql-explain)).

:exclamation: Внимательно отнеситесь к тому, как [публиковать информацию](#2.3-data-publication),
которая требуется для ответа на ваш вопрос.

Подробнее о важных сведениях по существу вопроса см. [тут](https://wiki.postgresql.org/wiki/Slow_Query_Questions/ru).

### <a name='2.5-invalid-query' /> [2.5.](#2.5-invalid-query) У меня не получается написать SQL запрос/etc и/или я получаю ошибки. Какую информацию мне необходимо предоставить для получения помощи от сообщества?

1. Кратко опишите предметную область и то, что вы хотите сделать;
2. Продемонстрируйте то, что вы уже сделали. Это докажет то, что вы попытались решить проблему самостоятельно,
а также даст начальную точку для участников сообщества, которые захотят вам помочь. Если вы получаете ошибки, то их также стоит приложить к своему вопросу;
3. Выполните хотя бы один из пунктов:<br>
    3.1. Предоставьте вывод метакоманды \d из утилиты [psql](https://postgrespro.ru/docs/postgresql/15/app-psql) для каждой таблицы, которая будет участвовать в запросе;<br>
    3.2. Создайте тестовое окружение, которое воспроизводит ваши [таблицы](https://postgrespro.ru/docs/postgresql/15/ddl)/данные и непосредственно проблему.
         Это поможет другим участникам сообщества легко и быстро приступить к изучению вашей проблемы (и, как следствие, повысит желание помогать в её решении).
         Для этого следует использовать такие сервисы как [sqlize.online](https://sqlize.online/) или [db-fiddle.com](https://www.db-fiddle.com/).<br>

:exclamation: Внимательно отнеситесь к тому, как [публиковать информацию](#2.3-data-publication),
которая требуется для ответа на ваш вопрос.

### <a name='2.6-register-codestyle' /> [2.6.](#2.6-register-codestyle) В каком регистре писать команды/функции/имена/etc в PostgreSQL?

Стоит отметить, что у сообщества нет единого мнения по данному вопросу.
Единственный технический нюанс, который следует учитывать, описан [выше](#2.2-name-does-not-exist).
В остальном, это зависит исключительно от того, как принято в вашей команде и/или организации (т.н. [code style](https://ru.wikipedia.org/wiki/%D0%A1%D1%82%D0%B0%D0%BD%D0%B4%D0%B0%D1%80%D1%82_%D0%BE%D1%84%D0%BE%D1%80%D0%BC%D0%BB%D0%B5%D0%BD%D0%B8%D1%8F_%D0%BA%D0%BE%D0%B4%D0%B0)).
Внутри команды и/или организации важно соблюдать единый стандарт оформления кода для того чтобы в дальнейшем его было легко читать/поддерживать как вам, так и вашим коллегам.

Если обратиться к примерам [официальной документации](https://www.postgresql.org/docs/) PostgreSQL (которые также имеют некоторое разночтение), то, как правило, им свойственно следующее:
1. Все [ключевые слова](https://postgrespro.ru/docs/postgresql/15/sql-keywords-appendix),
включая слова из [DML](https://ru.wikipedia.org/wiki/Data_Manipulation_Language) (_SELECT/INSERT/UPDATE/DELETE_), 
[DDL](https://postgrespro.ru/docs/postgresql/15/ddl) (_CREATE/ALTER/DROP_)
а также [DCL](https://ru.wikipedia.org/wiki/Data_Control_Language) (_GRANT/REVOKE_) пишутся в верхнем регистре. Например:
    ```sql
    SELECT * FROM table;
    ```
2. Идентификаторы (имена таблиц, столбцов, функций и т.д.) в большинстве случаев пишутся в нижнем регистре в формате [snake_case](https://ru.wikipedia.org/wiki/Snake_case). Например:
    ```sql
    SELECT floor(col_name) FROM table;
    ```


## <a name='3-learning' /> [3.](#3-learning) Книги/курсы :blue_book:
1. [Postgres: первое знакомство](https://www.postgrespro.ru/education/books/introbook) ([.pdf](https://edu.postgrespro.ru/introbook_v8.pdf));
2. [PostgreSQL изнутри](https://www.postgrespro.ru/education/books/internals) ([.pdf](https://edu.postgrespro.ru/postgresql_internals-14.pdf));
3. [Базовый курс DBA1](https://postgrespro.ru/education/courses/DBA1) от компании Postgres Professional.


## <a name='4-links' /> [4.](#4-links) Полезные ссылки :link:

1. [Документация PostgreSQL на русском языке](https://postgrespro.ru/docs/postgresql/);
2. [Официальный FAQ на русском языке](https://wiki.postgresql.org/wiki/FAQ/ru);
3. [Вредные и/или опасные действия в PostgreSQL](https://wiki.postgresql.org/wiki/Don't_Do_This) _(eng)_;
4. [Упражнения по SQL](https://www.sql-ex.ru/?Lang=0);
5. [Руководства по PostgreSQL, примеры использования](https://www.postgresqltutorial.com/) _(eng)_.
