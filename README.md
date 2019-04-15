# Тестовое задание 

## Серверная часть 

Серверная часть проекта решает задачу регистрации ключей на стороне сервера и управления ими через административную часть сервера.
 
Сервер использует базу данных для хранения ключей, в данном случае была выбрана [PostgreSQL](https://www.postgresql.org/), в качестве ORM системы для работы с БД используется [Gorm](http://gorm.io/).

Сервер написан на языке программирования [Go](https://golang.org/), c использованием [модульной системы](https://github.com/golang/go/wiki/Modules) для загрузки зависимостей.
Список зависимостей можно увидеть в файле: [go.mod](go.mod)

Операции получения процента загрузки ЦП реализованы для ОП Linux. В windows это работать не будет.

### Серверная часть разделена на 3 части: 

1. **Web-сервер** для интерфейса управления поступившими запросами. Описание доступных url адресов страниц см. в таблице Web server.
2. **RestApi** для получения сессии авторизации и выхода из сессии(табл. RestAPi).
3. **WebSocket** подключения, одно для подключения клиента. И ещё одно для администратора для работы со списком запросов(табл. Websocket).

В рамках данной задачи для удобства все эти сервисы запускаются одним исполняемым файлом.

### Запуск сервера: 

Для осуществления работы с базой данных, используется [docker](https://www.docker.com/).
Перед запуском сервера необходимо запустить докер.

#### Запуск докера
Как установить и запустить Docker, возможно при первом запуске придётся создать базу данных `main` и таблицу `keys` см. ниже SQL запрос на создание таблицы:
 1. [Установка Docker-CE (ubuntu)](https://docs.docker.com/install/linux/docker-ce/ubuntu/)
 2. [Установка Docker compose](https://docs.docker.com/compose/install/)
 3. В папке `docker` запустить: `sudo docker-compose up`

После запуска докера запускаем сервер, до процесса компиляции он находится в папке: `/server` файл `main.go` (go run main.go)

Для конфигурирования сервера, используется файл settins.json (json формат [RFC7159](https://tools.ietf.org/html/rfc7159)), 
данный файл должен находится в той же директории где находится исполняемый файл сервера.

Формат файла настроек settins.json, все поля в данном файле обязательны, из название поля должно быть понятно за что оно отвечает, секция gorm отвечает за подключение к БД:

```json 
{
  "name": "api",
  "version": "1.0.0",
  "host" : "localhost",
  "port" : "8080",
  "gorm" : {
    "type" : "postgres",
    "host" : "localhost",
    "port" : "45432",
    "user" : "postgres",
    "password" : "postgres",
    "database" : "main",
    "connPattern" : "%s://%s:%s@%s:%s?database=%s&sslmode=disable"
  }
} 
```

### Таблицы в БД 

Для работы сервиса необходима 1 таблица `keys` SQL запрос для создания таблицы:

```sql
-- auto-generated definition
create table keys
(
	id serial
		constraint keys_pk
			primary key,
	key varchar(256),
	name varchar(256),
	ip varchar(256),
	status SMALLINT
);

create unique index keys_key_uindex
	on keys (key);
```

#### Описание сущности в таблице keys: 

Ключ | Тип | Значение
---|---|--- 
id |int|Уникальный идентификатор ключа, создается при добавлении ключа в таблице 
key |string|Ключ который передал клиент формат: **xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx** 
name|string|Имя ключа, при первичной регистрации заполняется UserAgent клиента, далее администратор в праве задать имя самостоятельно.
ip|string|Ip клиента при отправки ключа.
status|int|Статус ключа. См. Ниже описание статусов.

#### Описание статусов ключа: 

Статус | Описание
---|--- 
0 |Статус не назначен, присваивается при первичном обращении клиента.
1 |Статус активен, т.е. может создать websocket подключение и получать статистику загрузки ЦП.
2 |Заблокирован. При обращении с этим ключом, будет возвращать статус 1008 и сообщение "banned"

#### Дополнительные статусы(в БД не заносятся) 

Статус | Описание
---|---
4 |Данный статус отправляется в запросе на получение списка ключей администратором. Создан для того что бы получить все возможные ключи из БД в секции "Все".
100 |Онлайн, отправляется клиентам в момент создания подключения ключа с активным статусом. Что бы администраторы могли его отобразить в секции online.
200 |Офлайн, отправляется клиентом когда активный клиент отключился, по собственной причине или по причине блокировки в момент его активного подключения. Для того что бы администраторы если у них выбрана секция online могли убрать его из списка.

### Таблицы для описания подключений:

#### Web server

Страница управление запросами и клиент написаны с использованием [vue.js](https://vuejs.org/). 
Для формы авторизации используется [jquery 3.3](https://jquery.com/).

Публичная часть сервера(css, js, images и т.д.) находится в папке `server/http_files/public`

Статика для url(html страницы) хранится: `server/http_files/` пользователям не получить к ним доступ напрямую.

Url  | Описание
---|--- 
/ | основная страница сервера, если пользователь авторизован то переходит на страницу управления запросов. Иначе будет отображена форма авторизации.
/client | веб клиент для подключения к серверу. Также можно открывать: server/http_files/client.html в браузере, и тоже будет работать.

#### RestAPi
 Url | Описание
---|---
/login | Авторизация, отправляем логин и пароль POST запросом для авторизации(переменные: login, pass). Логин: admin Пароль: admin
/logout | Разлогинится.

#### Websocket
  
 Url | Описание
---|--- 
/st_cpu | Отправить ключ, и уже после одобрения в админке ключа можно подключиться и получать загрузку CPU.
/ws_list | Подключение для работы со списком запросов. Необходима авторизация.
    
### Описание работы административной части

Представляет собой веб интерфейс доступ к которому защищен логином и паролем:

![login form](/images/admin_login.png)

Логин: **admin** Пароль: **admin** Для реализации обновления данных в реальном времени, административная часть используется подключение по **websocket**(адрес: /ws_list, для подключения необходима авторизация).
Веб-интерфейс для управления запросами находится по адресу: **http://localhost:8080/** Если пользователь успешно пройдет авторизацию то его перекинет на страницу управления запросами.
Для удобства сортировки и управления ключами, интерфейс разделен на 5 категорий:

![list](/images/admin_list.png)

1. **Запросы** в эту категорию попадают ключи, которым администратор ещё не присвоил статус.
2. **Разрешенные** список ключей с которым администратор разрешил доступ.
3. **Заблокированные** список заблокированных ключей.
4. **Все** список всех ключей.
5. **Online** список ключей которым администратор назначил статус активен, и которые в данный момент подключены к серверу.

В каждой категории администратор имеет права назначить статус ключу. Но есть ограничений когда мы находимся к примеру в списке разрешенных ключей, мы не сможем ещё раз отметить ключ активным.
Все списки обновляются в реальном времени.

Т.е. если у администратора, открыт список запросов, и в данный момент появился новый запрос, он отобразится в этом списке. Или к примеру у 1 администратора открыт список разрешенных ключей, а 2 администратор из списка заблокированных ключей перенесет ключ в статус активных, этот ключ без обновления страницы отобразится у 1 администратора.

В момент присвоения нового статуса ключу администратор так же может присвоить ключу имя, во всплывающем диалоговом окне.

![popup](/images/admin_popup.png)

### Описание работы клиента который отправляет запрос с ключом:

Клиент можно запустить при запущенном сервере по адрсесу: <http://localhost:8080/client> или открыть в браузере страницу из папки: `server/http_files/client.html` что менее предпочтительно.

Клиент осуществляет доступ к серверу через **websoket** подключение _/st_cpu_  
В желтом окошечке у клиента ведётся лог подключения он сессионный, то есть если закрыть полностью окно браузере он сотрётся. Это сделано из-за ограничения размера cookies.

При запуске клиента генерируется ключ, если до этого был сгенерирован загружается уже сгенерированый ключ.

![client](/images/client_firstconn.png)

После того когда клиент введёт адрес сервера, он может нажать кнопку: "Установить соединение", если клиент ещё не отправлял ключ, 
то он отправит свой ключ, сервер его примет(и занесёт в таблице `keys`) и пока администраторы не одобрят его клиент будет 
получать сообщение со статусом `1000` и текстом `Key accepted, please wait and connected again.`

Если администраторы одобрили ключ то клиент при следующем нажатии на кнопку: "Установить соединение" сможет успешно подключится и получать статистику загрузки ЦП сервера.

![client](/images/client_connected.png)

Если администраторы забанят ключ то клиент будет получать сообщение со статусом `1008` и текстом `banned`. 
Даже если у клиента одобрили ключ его могут забанить в любой момент, даже в момент активного соединения, в этом случае клиент получит сообщение о бане и сервер закроет его соединение.

![client](/images/client_banned.png)

## Описание задания

Написать сервер и клиента со следующим функционалом:
1. При запуске клиента ему передаётся адрес сервера. Клиент формирует уникальный ключ и пытается подключиться с ним к серверу.
2. Сервер имеет простой web-интерфейс, закрытый паролем.
3. В интерфейсе сервера можно посмотреть все запросы от клиентов и разрешить или отклонить запрос.
4. Если запрос принят, то секретный ключ клиента сохраняется в список разрешённых и клиент может поднять web-сокет для обмена состоянием с сервером.
5. Если запрос отклонён, то секретный ключ клиента сохраняется в список запрещённых и клиент больше не отображается списке запросов.
6. Клиент должен уметь переживать перезагрузку.
7. Сервер также должен показывать список разрешённых клиентов.
8. В информации о клиенте должны быть имя и IP адрес машины клиента.
9. У клиентов, подключенных по сокету необходимо показывать загрузку процессора.

Примечание:
В данном случае речь о взаимодействии сервисов, поэтому описываемое взаимодействие "клиент-сервис" по сути является взаимодействием "сервис-сервис"