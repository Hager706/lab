to start mysql:
mkdir ~/mysql-tablespace
- docker run --name my-mysql \
  -e MYSQL_ROOT_PASSWORD=123 \
  -p 3306:3306 \
  -v ~/mysql-data:/var/lib/mysql \
  -d mysql:8.0
  -v ~/mysql-tablespace:/datafile \
- docker exec -it my-mysql mysql -u root -p
                    |        |
                    ^        ^
            this container  this app mysql inside the container

- docker exec -it my-mysql bash

************************day3***************************
1-show databases;
2- use sys
      SHOW TABLES;
      select * from processlist\G
      select * from host_summary\G
      Select * from user_summary\G
      Select * from session\G >> show how login to db every user in row 

3-use mysql
      SHOW TABLES;
      select * from user\G >>
      mysql select * from user\G
            *** 1. row ***
            Host: %   ----> from any host 
            User: root

            *** 2. row ***
            Host: localhost  ----> from any host 
            User: mysql.infoschema

      select * from plugin\G
      use performance_schema
      use information_schema
      Select * from tables\G

4-create database db1;
      use db1
      to check>> ls /var/lib/mysql/db1
      Create table tblINNODB(ID int) engine=INNODB; >> default
      Create table tblMyISAM(ID int) engine=MYISAM;
      Create table tblMemory(ID int) engine=MEMORY;
      Create table tblCSV(ID int) engine=CSV;
      drop database db1;

************************day4***************************
create database db1;
use db1; 
prompt \d>>>

:InnoDB 
1️⃣ Table 
2️⃣ Tablespace 
3️⃣ Datafile (.ibd) → فايل فعلي على الديسك

1-create table Test1(id int);  > File-per-table tablespace (default)
   
CREATE TABLESPACE myspace add datafile 'myspace.ibd'; 

2-create table Test2(id int) TABLESPACE=myspace; 
هنعمل تيبل جديد بس بدل ميتحط جوا 
db1 > Test2.ibd 
لا هو هيتحط هوا النيم اسبيس 
myspace.ibd > Test2 data

mkdir /datafile  chown mysql /datafile
CREATE TABLESPACE mysecspace add datafile '/datafile/mysecspace.ibd';
CREATE TABLE Test3(id int) TABLESPACE=mysecspace;
CREATE TABLE Test4(id int) DATA DIRECTORY='/datafile';

Test1 <<<< db1/Test1.ibd 	db1/Test1.ibd → كل جدول جوه
Test2 <<<< myspace.ibd
Test3 <<<< /datafile/mysecspace.ibd
Test4 <<<< /datafile/Test4.ibd (لوحده)  → الجدول هيتحط جوا فولدر تاني عشان لو حابه افصله في فوليم او ديسك منفصل يعني 

show create table Test1\G;
show create table Test2\G;
show create table Test3\G;
show create table Test4\G;

************************day5 backup & restore ***************************
CREATE DATABASE testdb;
USE testdb;
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50),
);

INSERT INTO users (name) VALUES
('Hagar'),
('Ali'),

SELECT * FROM users;

**backup:**
1-mysqldump -u root -p testdb > /var/lib/mysql/testdb_backup.sql
2-mysqldump -u root -p --databases testdb1 testdb2 > /var/lib/mysql/multiple_db_backup.sql
3-mysqldump -u root -p --all-databases > /var/lib/mysql/all_db_backup.sql

**restore:**
1-in same container an db already exit 
   mysql -u root -p testdb < /var/lib/mysql/testdb_backup.sql

2-in same container an db not exit 
   CREATE DATABASE newdb;
   mysql -u root -p newdb < /var/lib/mysql/testdb_backup.sql

3- not same container 
- docker run --name my-mysql-new \
   -e MYSQL_ROOT_PASSWORD=123 \
   -p 3307:3306 \
   -v ~/mysql-data-new:/var/lib/mysql \
   -d mysql:8.0
 
- docker exec -it my-mysql-new bash
- mysql -u root -p < /var/lib/mysql/testdb_backup.sql

**************day6 Point in time recovery*******************
	Binary Log هو ملف يسجل كل التغييرات اللي بتحصل على قاعدة البيانات بعد تشغيل السيرفر.
يعني أي INSERT، UPDATE، DELETE، أو تغييرات على الجداول، كلها بتتسجل هناك بالترتيب اللي حصلت فيه.
	•	الهدف الأساسي منه:
	1.	Replication: لما يكون عندك أكثر من سيرفر MySQL، تقدر تزامنهم.
	2.	Point-in-Time Recovery (PITR): ترجعي الداتا لأي وقت محدد بعد الباكب.

1- # /etc/mysql/my.cnf 
[mysqld]
log-bin=mysql-bin
binlog_format=row
expire_logs_days=7

2- restart >> mysqladmin -u root -p shutdown
mysqld_safe &
3- CREATE DATABASE testdb;
USE testdb;
CREATE TABLE users (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(50));
INSERT INTO users (name) VALUES ('Hagar'), ('Ahmed');
4-mysqldump -u root -p testdb > /var/lib/mysql/testdb_backup.sql
5-INSERT INTO users (name) VALUES ('Ali');
DELETE FROM users WHERE name='Ahmed';
6-mysql -u root -p testdb < /var/lib/mysql/testdb_backup.sql
7-SHOW BINARY LOGS;
-mysqlbinlog /var/lib/mysql/mysql-bin.000001 | less
-mysqlbinlog --stop-datetime="2026-01-20 11:10:00" /var/lib/mysql/mysql-bin.000001 | mysql -u root -p testdb

************************day7  Users in MySQL***************************
CREATE USER 'username'@'host' IDENTIFIED BY 'password';

SELECT user, host FROM mysql.user;

GRANT ALL PRIVILEGES ON testdb.* TO 'app_user'@'%';

GRANT SELECT, INSERT, UPDATE ON testdb.* TO 'app_user'@'%';

GRANT SELECT ON testdb.users TO 'app_user'@'%';

GRANT ALL PRIVILEGES ON *.* TO 'admin_user'@'localhost';

FLUSH PRIVILEGES;

SHOW GRANTS FOR 'app_user'@'%';
mysql -u app_user -p    ||  mysql -u app_user -p -h 127.0.0.1 -P 3306
ex : CREATE USER 'backup_user'@'localhost' IDENTIFIED BY 'Backup@123';
GRANT SELECT, LOCK TABLES, SHOW VIEW ON *.* TO 'backup_user'@'localhost';

**(REVOKE)**
REVOKE DELETE ON testdb.* FROM 'app_user'@'%';

DROP USER 'app_user'@'%';
************************day8***************************
لو نزلت داتا بيز وهي عندي ممكن عشان اخليها تيجي علي الداتا بيز بتاعتي 
source /bath of download db

SET GLOBAL general_log = 'ON'; 
SHOW VARIABLES LIKE 'general_log_file';
SET GLOBAL general_log = 'OFF';
EXPLAIN SELECT * FROM users WHERE name = 'Hagar';

SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 2;
SHOW VARIABLES LIKE 'slow_query_log_file';

SHOW PROCESSLIST;
SELECT * FROM information_schema.INNODB_TRX;

************************day8 blocking***************************
Blocking =
Transaction ماسكة Lock
Transaction تانية جاية تشتغل على نفس الداتا
→ فتقف تستنى
ليه؟
عشان Data Consistency
MySQL (InnoDB) مينفعش يسيب اتنين يغيروا نفس الصف في نفس الوقت.



SHOW ENGINE INNODB STATUS\G