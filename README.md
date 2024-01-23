
# Домашнее задание к занятиям «Репликация и масштабирование»

### Инструкция по выполнению домашнего задания

### Задание 1

Выполните конфигурацию master-slave репликации, примером можно пользоваться из лекции.

*Приложите скриншоты конфигурации, выполнения работы: состояния и режимы работы серверов.*

---

### Задание 2

Разработайте план для выполнения горизонтального и вертикального шаринга базы данных. База данных состоит из трёх таблиц: 

- пользователи, 
- книги, 
- магазины (столбцы произвольно). 

Опишите принципы построения системы и их разграничение или разбивку между базами данных.

*Пришлите блоксхему, где и что будет располагаться. Опишите, в каких режимах будут работать сервера.* 


---

### Решение 1

Подготовка контейнеров Master и Slave  MySQL

```
	docker-compose up -d
	docker-compose ps
	docker exec -it mysql-master bash
	docker exec -it mysql-slave bash
```

Скриншот запуска двух БД MySQL в Docker 

![Commit Task1](https://github.com/AndrewZnamenskiy/Replication-and-Scaling/blob/main/img/task1p1.png)


Залогиниваемся в БД Master

	mysql -u root -prootsecret
 
Заведение пользователя для репликации

```
	CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'pass123';
	GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
	SELECT User FROM mysql.user;
```

Скриншот заведения пользователя repl

![Commit Task1](https://github.com/AndrewZnamenskiy/Replication-and-Scaling/blob/main/img/task1p2.png)


Создаём БД test1 на Master

	CREATE DATABASE test1;

Скриншот заведения тестовой БД

![Commit Task1](https://github.com/AndrewZnamenskiy/Replication-and-Scaling/blob/main/img/task1p3.png)


Для начала репликации проверяем статус мастера и фиксируем имя файла и позицию

	`SHOW MASTER STATUS;`

Далее блокируем все изменения на сервере мастер

	`FLUSH TABLES WITH READ LOCK;` 
 
Экспортируем базу Master для переноса на Slave
 
	`docker exec mysql-master mysqldump -u root -prootsecret test1 > /home/andy/MySQL/test1.sql`

Снимаем блокировку с Master базы

	`UNLOCK TABLES;`
 
Вручную переносим базу Master на Slave:

- копируем базу на Slave

	`docker cp /home/andy/MySQL/test1.sql mysql-slave:/tmp/test1.sql`

- создаём базу 'test1' на Slave

	`docker exec -it mysql-slave mysql -u root -prootsecret`

	`CREATE DATABASE test1;`

- импортируем базу 'test1' на Slave

	`docker exec -it mysql-slave bash`

	`mysql -u root -prootsecret test1 < /tmp/test1.sql`


Скриншот переноса и импорт базы test1 на Slave

![Commit Task1](https://github.com/AndrewZnamenskiy/Replication-and-Scaling/blob/main/img/task1p4.png)


![Commit Task1](https://github.com/AndrewZnamenskiy/Replication-and-Scaling/blob/main/img/task1p5.png)



Запуск репликации на Slave:

```
	STOP SLAVE;

	CHANGE MASTER TO 
	MASTER_HOST='mysql-master',
	MASTER_USER='repl',
	MASTER_PASSWORD='pass123', 
	MASTER_LOG_FILE='mysql-bin.000003',
	MASTER_LOG_POS=848;

	START SLAVE;
	SHOW SLAVE STATUS\G;
```

Скриншот успешной репликации Master-Slave

![Commit Task1](https://github.com/AndrewZnamenskiy/Replication-and-Scaling/blob/main/img/task1p6.png)



### Решение 2

Скриншот вертикального шардирования 

![Commit Task2](https://github.com/AndrewZnamenskiy/Replication-and-Scaling/blob/main/img/task2p1.png)


Скриншот горизонтального шардирования

![Commit Task2](https://github.com/AndrewZnamenskiy/Replication-and-Scaling/blob/main/img/task2p2.png)


Вертикальне шардирование в данном случае выполнятся путём разнесения таблиц исходной БД на 
три отдельные БД. Горизонтальное шардирование выполняется путём распредления части строк БД на БД1, 
а другой части на БД2. Разбиение таблиц может  происходить по диапазонам значений одного из столбцов, 
по признаку чётности и т.д. В горизонтальном шардировании все три таблицы располагаются на БД1 и БД2, 
т.е. шардах исходной БД. Базы в обоих вариантах работают в режиме "Мастер", а объединение данных 
осуществляется на стороне клиента.


