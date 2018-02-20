###Comandos generales {#commands}

* Instalar MySQL server en Centos 

    [root@server ~]# yum install mysql-server

* Cambiar el password de root cuando recién se instala el MySQL server
Ni bien se instala el MySQL server, la cuenta de root no tiene password seteado. Ésto, se setea de la siguiente manera:

    `[root@server ~]# mysqladmin -u root password 'new-password'`

* Búsquedas de entradas en tablas 

```
mysql> select calldate,src,dst,duration from asteriskcdrdb.cdr where calldate like '2014-01-12%';
mysql> select calldate,src,dst,duration from asteriskcdrdb.cdr where calldate like '2013-11-30%' and dst like '0341154700251';
mysql> select calldate,src,dst,duration from asteriskcdrdb.cdr where calldate like '2014-01-11%' and src like '0221156125249';
mysql> select * from asteriskcdrdb.cdr where calldate like '2014-07-30%' and dst like '9086' and dstchannel like 'Local/176%';
```

* Eliminar base de datos completa 
```
mysql> drop database qstats;
Query OK, 16 rows affected (17.10 sec)
```

* Restore de base de datos 

```
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| test               |
+--------------------+
3 rows in set (0.03 sec)

mysql> create database queuemetrics;
Query OK, 1 row affected (0.05 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| queuemetrics       |
| test               |
+--------------------+
4 rows in set (0.00 sec)

[root@ftsender ~]# mysql -u root -p queuemetrics < 20161006-queuemetrics.sql
```

* Duplicar tabla de base de datos 

```
mysql> show tables;
+-------------------------+
| Tables_in_queuemetrics  |
+-------------------------+
| agaw_alarms             |
| agaw_dati               |
| agaw_logs               |
| agaw_runs               |
| agent_groups            |
| agent_history           |
| agenti_noti             |
| arch_classes            |
| arch_syslog             |
| arch_users              |
| broadcast_msg           |
| call_features           |
| call_status             |
| call_tags               |
| code_possibili          |
| dbversion               |
| dnis                    |
| export_calls            |
| export_jobs             |
| export_reports          |
| fb_actions              |
| ivr                     |
| known_numbers           |
| locations               |
| oq_calls                |
| oq_notes                |
| oq_queues               |
| pause_codes             |
| prl_notes               |
| qa_comments             |
| qa_data                 |
| qa_forms                |
| qa_forms_items          |
| qa_forms_items_attr     |
| qa_perftrack_rulegroups |
| qa_perftrack_rules      |
| qlog_opencalls          |
| qm_cbts                 |
| qm_queries              |
| qm_reports              |
| qm_tasks                |
| queue_log               |
| record_tags             |
| skills                  |
| user_prefs              |
+-------------------------+
45 rows in set (0.00 sec)

mysql> create table queue_log_b like queue_log;
Query OK, 0 rows affected (0.09 sec)

mysql> show tables;
+-------------------------+
| Tables_in_queuemetrics  |
+-------------------------+
| agaw_alarms             |
| agaw_dati               |
| agaw_logs               |
| agaw_runs               |
| agent_groups            |
| agent_history           |
| agenti_noti             |
| arch_classes            |
| arch_syslog             |
| arch_users              |
| broadcast_msg           |
| call_features           |
| call_status             |
| call_tags               |
| code_possibili          |
| dbversion               |
| dnis                    |
| export_calls            |
| export_jobs             |
| export_reports          |
| fb_actions              |
| ivr                     |
| known_numbers           |
| locations               |
| oq_calls                |
| oq_notes                |
| oq_queues               |
| pause_codes             |
| prl_notes               |
| qa_comments             |
| qa_data                 |
| qa_forms                |
| qa_forms_items          |
| qa_forms_items_attr     |
| qa_perftrack_rulegroups |
| qa_perftrack_rules      |
| qlog_opencalls          |
| qm_cbts                 |
| qm_queries              |
| qm_reports              |
| qm_tasks                |
| queue_log               |
| queue_log_b             |
| record_tags             |
| skills                  |
| user_prefs              |
+-------------------------+
46 rows in set (0.00 sec)

mysql> insert into queue_log_b select * from queue_log;
Query OK, 3290056 rows affected (2 min 57.68 sec)
Records: 3290056  Duplicates: 0  Warnings: 0

mysql> select count(*) from queue_log;
+----------+
| count(*) |
+----------+
|  3290056 |
+----------+
1 row in set (0.05 sec)

mysql> select count(*) from queue_log_b;
+----------+
| count(*) |
+----------+
|  3290056 |
+----------+
1 row in set (0.00 sec)
```
---
###Crear usuario para conectarse remotamente {#userremote}

In order to connect remotely you have to have MySQL bind port 3306 to your machine's IP address in `my.cnf`. 
Then you have to have created the user in both localhost and '%' wildcard and grant permissions on all DB's as such. 
See below: `my.cnf` (my.ini on windows)

```
#Replace xxx with your IP Address 
bind-address        = xxx.xxx.xxx.xxx

then

CREATE USER 'myuser'@'localhost' IDENTIFIED BY 'mypass';
CREATE USER 'myuser'@'%' IDENTIFIED BY 'mypass';

Then

GRANT ALL ON . TO 'myuser'@'localhost';
GRANT ALL ON . TO 'myuser'@'%';
```
---
###Reparar todas las bases de datos {#repairdb}

    mysqlcheck -u root -p --repair --all-databases

---
