# 12_06_replication_01

Задание 1  
  
На лекции рассматривались режимы репликации master-slave, master-master, опишите их различия.  

Режимы репликации в MySQL, такие как master-slave и master-master, предоставляют возможность создания резервных копий данных, повышения отказоустойчивости и распределения нагрузки между базами данных. Основные различия:  

Режим репликации master-slave  

В режиме master-slave имеется один главный сервер (мастер) и один или несколько вспомогательных серверов (слейвы).  
Мастер-сервер является источником данных, которые реплицируются на слейв-серверы.  
Слейв-серверы могут использоваться для чтения данных и обеспечения отказоустойчивости.  
Мастер-сервер обрабатывает запись данных, а слейв-серверы реплицируют эти данные из мастера.  
Слейв-серверы работают в режиме только для чтения и не принимают записи.  
Режим master-slave обеспечивает резервное копирование данных, но не позволяет обновлять данные на слейв-серверах.  
Режим репликации master-master:  
  
В режиме master-master имеется несколько мастер-серверов, каждый из которых может обрабатывать записи и чтение данных.  
Каждый мастер-сервер может принимать и обновлять данные независимо от других мастер-серверов.    
Репликация данных происходит в обоих направлениях между мастер-серверами  
Режим master-master позволяет балансировать нагрузку и обеспечивает отказоустойчивость  
Однако необходимо обращать особое внимание на разрешение конфликтов при записи данных на разных мастер-серверах.  
Режимы репликации master-slave и master-master имеют свои особенности и применяются в различных сценариях. Master-slave обычно используется для резервного копирования данных и обеспечения отказоустойчивости, в то время как master-master может быть полезен для балансировки нагрузки и обеспечения высокой доступности данных.  
      
Ответить в свободной форме.    
Задание 2   
  
Выполните конфигурацию master-slave репликации, примером можно пользоваться из лекции.  
  
Приложите скриншоты конфигурации, выполнения работы: состояния и режимы работы серверов.



Установка master:  
docker run -d --name replication-master -e MYSQL_ALLOW_EMPTY_PASSWORD=true -v /path/to/world/dump:/docker-entrypoint-initdb.d mysql:8.0-debian  
  
Установка slave:  
docker run -d --name replication-slave -e MYSQL_ALLOW_EMPTY_PASSWORD=true mysql:8.0-debian  

Для реализации взаимодействия создадим мост и сеть:  

docker network create replication  
docker network connect replication replication-master  
docker network connect replication replication-slave  

Для управления реализации настроек обновим и установим в контейнеры инструменты.  
Для Master:  

docker exec replication-master apt-get update && docker exec replication-master apt-get install -y nano  

Для Slave:  

docker exec replication-slave apt-get update && docker exec replication-slave apt-get install -y nano  

Создадим учетную запись Master для сервера репликации:  

docker exec -it replication-master mysql  
В контейнере выполним:  
mysql> CREATE USER 'replication'@'%';  
mysql> GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';  
Изменим конфигурацию сервера:  
docker exec -it replication-master bash nano /etc/mysql/my.cnf  
my.cnf -> секция [mysqld]   
server_id = 1  
log_bin = mysql-bin  
При изменении конфигурации сервера требуется перезагрузка:  
docker restart replication-master  
После требуется зайти в контейнер и проверить состояние:  
docker exec -it replication-master mysql  
mysql> SHOW MASTER STATUS;  

Следующим шагом требуется выполнить слепок системы и заблокировать все изменения на сервер:  
mysql> FLUSH TABLES WITH READ LOCK;  
После данных манипуляций выхода из контейнера и выполняем процесс mysqldump для экспорта базы данных, например:  
docker exec replication-master mysqldump world > /path/to/dump/on/host/world.sql  
После, следует зайти обратно в контейнер и вывести настройки master сервера (они понадобятся при настройке slave):  
docker exec -it replication-master mysql  
mysql> SHOW MASTER STATUS  

Снимаем блокировку базы данных:  
mysql> UNLOCK TABLES;  
Master готов, переходим к slave:  
docker cp /path/to/dump/on/host/world.sql replication-slave:/tmp/world.sql  
docker exec -it replication-slave mysql  
mysql> CREATE DATABASE `world`;  
docker exec -it replication-slave bash mysql world < /tmp/world.sql  
docker exec -it replication-slave bash nano /etc/mysql/my.cnf  

log_bin = mysql-bin  
server_id = 2  
relay-log = /var/lib/mysql/mysql-relay-bin  
relay-log-index = /var/lib/mysql/mysql-relay-bin.index  
read_only = 1  
docker restart replication-slave  

Следующим шагом требуется прописать в базе данных на сервер slave, кто является master и данные полученные в File и Position:  

docker exec -it replication-slave mysql  
mysql> CHANGE MASTER TO MASTER_HOST='replication-master',  
MASTER_USER='replication', MASTER_LOG_FILE='mysql-bin.000001',  
MASTER_LOG_POS=156;  
mysql> START SLAVE;  
mysql> SHOW SLAVE STATUS\G  
Далее запускаем журнал ретрансляции, и проверим статус  

<img src = "img/master.png" width = 100%>  
<img src = "img/slave.png" width = 100%>  
<img src = "img/master_conf.png" width = 100%>  
<img src = "img/slave_conf.png" width = 100%>  
