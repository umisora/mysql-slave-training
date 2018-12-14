# Step 1
```
git clone git@github.com:umisora/mysql-slave-training.git
cd mysql-slave-training
docker-compose up
```

接続確認
```
mysql -u user -ppassword -P 13300 -h 127.0.0.1 
mysql -u user -ppassword -P 13310 -h 127.0.0.1
```

一度落としてSlaveをOnにする
```
docker-compose stop
```

Binlogを有効化する
```
@master01
> vi master01/conf.d/master.cnf 

[mysqld]
+ server-id=1
+ gtid_mode=ON
+ enforce-gtid-consistency=true
+ log_bin=mysql-bin
+ sync_binlog=1
```

```
@slave01
vi slave01/conf.d/slave.cnf

[mysqld]
+ iserver-id=900
+ gtid_mode=ON
+ enforce-gtid-consistency=true
+ log_bin=mysql-bin
+ sync_binlog=1
+ skip-slave-start
+ log_slave_updates
+ read_only
+ relay_log=mysql-relay-bin
```

Replication Userの作成
```
@master01
mysql> CREATE USER 'repl'@'%' IDENTIFIED BY 'password';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
mysql> commit;
```

テストデータ入れてみる
```
wget http://downloads.mysql.com/docs/world.sql.gz
gunzip world.sql.gz

@master01
mysql -u root -ppassword -P 13300 -h 127.0.0.1 < world.sql
mysql -u root -ppassword -P 13300 -h 127.0.0.1 world


mysql> select * from city order by ID desc limit 3;
+------+----------+-------------+------------+------------+
| ID   | Name     | CountryCode | District   | Population |
+------+----------+-------------+------------+------------+
| 4079 | Rafah    | PSE         | Rafah      |      92020 |
| 4078 | Nablus   | PSE         | Nablus     |     100231 |
| 4077 | Jabaliya | PSE         | North Gaza |     113901 |
+------+----------+-------------+------------+------------+
3 rows in set (0.00 sec)

mysql> insert into city (Name,CountryCode,District,Population) values ('umisora','JPN','umisora',9999);
Query OK, 1 row affected (0.01 sec)

mysql> select * from city order by ID desc limit 3;
+------+---------+-------------+----------+------------+
| ID   | Name    | CountryCode | District | Population |
+------+---------+-------------+----------+------------+
| 4080 | umisora | JPN         | umisora  |       9999 |
| 4079 | Rafah   | PSE         | Rafah    |      92020 |
| 4078 | Nablus  | PSE         | Nablus   |     100231 |
+------+---------+-------------+----------+------------+
3 rows in set (0.01 sec)


mysql> show master status;
+------------------+----------+--------------+------------------+-------------------------------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                         |
+------------------+----------+--------------+------------------+-------------------------------------------+
| mysql-bin.000003 |   691413 |              |                  | 41fcd501-ff66-11e8-b716-0242ac130003:1-24 |
+------------------+----------+--------------+------------------+-------------------------------------------+
1 row in set (0.00 sec)
```

Slaveを作る(dump編)
```
# MasterからDump取得する
mysqldump -u root -p -h 127.0.0.1 -P 13300 --single-transaction --flush-logs --master-data --all-databases --column-statistics=0 > master.sql
mysql -u root -p -h 127.0.0.1 -P 13310 -e "RESET MASTER"
mysql -u root -p -h 127.0.0.1 -P 13310 < master.sql
mysql -u root -p -h 127.0.0.1 -P 13310 world
select * from city order by ID desc limit 3;
```

Slaveを開始する
```
mysql> CHANGE MASTER TO MASTER_HOST="mysql-training-master01", MASTER_PORT=3306, MASTER_USER='repl', MASTER_PASSWORD='password', MASTER_AUTO_POSITION=1;

mysql> show slave status \G
*************************** 1. row ***************************
               Slave_IO_State:
                  Master_Host: mysql-training-master01
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File:
          Read_Master_Log_Pos: 4
               Relay_Log_File: mysql-relay-bin.000001
                Relay_Log_Pos: 4
        Relay_Master_Log_File:
             Slave_IO_Running: No 
            Slave_SQL_Running: No
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 0
              Relay_Log_Space: 154
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 0
                  Master_UUID:
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State:
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set: 41fcd501-ff66-11e8-b716-0242ac130003:1-24
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)

mysql> start slave;
Query OK, 0 rows affected (0.01 sec)

mysql> show slave status \G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: mysql-training-master01
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000005
          Read_Master_Log_Pos: 194
               Relay_Log_File: mysql-relay-bin.000002
                Relay_Log_Pos: 367
        Relay_Master_Log_File: mysql-bin.000005
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 194
              Relay_Log_Space: 574
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 1
                  Master_UUID: 41fcd501-ff66-11e8-b716-0242ac130003
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set: 41fcd501-ff66-11e8-b716-0242ac130003:1-24
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)

```

# Slave動作確認
```
@master01
mysql -u root -ppassword -P 13300 -h 127.0.0.1 world
mysql> insert into city (Name,CountryCode,District,Population) values ('umisora','JPN','umisora',9999);
Query OK, 1 row affected (0.00 sec)

mysql> select * from city order by ID desc limit 3;
+------+---------+-------------+----------+------------+
| ID   | Name    | CountryCode | District | Population |
+------+---------+-------------+----------+------------+
| 4081 | umisora | JPN         | umisora  |       9999 |
| 4080 | umisora | JPN         | umisora  |       9999 |
| 4079 | Rafah   | PSE         | Rafah    |      92020 |
+------+---------+-------------+----------+------------+
3 rows in set (0.00 sec)


@slave01
mysql -u root -ppassword -P 13310 -h 127.0.0.1 world
mysql> select * from city order by ID desc limit 3;
+------+---------+-------------+----------+------------+
| ID   | Name    | CountryCode | District | Population |
+------+---------+-------------+----------+------------+
| 4081 | umisora | JPN         | umisora  |       9999 |
| 4080 | umisora | JPN         | umisora  |       9999 |
| 4079 | Rafah   | PSE         | Rafah    |      92020 |
+------+---------+-------------+----------+------------+
3 rows in set (0.00 sec)
```

# テーブル単位で除外
world.countrylanguage を レプリケーション対象外にする。
現状はレプリケーションされた状態

## まずは状態確認
```
@master01
mysql> select * from countrylanguage where CountryCode="JPN";
+-------------+----------------------+------------+------------+
| CountryCode | Language             | IsOfficial | Percentage |
+-------------+----------------------+------------+------------+
| JPN         | Ainu                 | F          |        0.0 |
| JPN         | Chinese              | F          |        0.2 |
| JPN         | English              | F          |        0.1 |
| JPN         | Japanese             | T          |       99.1 |
| JPN         | Korean               | F          |        0.5 |
| JPN         | Philippene Languages | F          |        0.1 |
+-------------+----------------------+------------+------------+
7 rows in set (0.00 sec)

@slave01
mysql> select * from countrylanguage where CountryCode="JPN";
+-------------+----------------------+------------+------------+
| CountryCode | Language             | IsOfficial | Percentage |
+-------------+----------------------+------------+------------+
| JPN         | Ainu                 | F          |        0.0 |
| JPN         | Chinese              | F          |        0.2 |
| JPN         | English              | F          |        0.1 |
| JPN         | Japanese             | T          |       99.1 |
| JPN         | Korean               | F          |        0.5 |
| JPN         | Philippene Languages | F          |        0.1 |
+-------------+----------------------+------------+------------+
7 rows in set (0.00 sec)

@master01
mysql> insert into countrylanguage values ('JPN',"MofuMofu","T","9.99");
Query OK, 1 row affected (0.01 sec)

mysql> select * from countrylanguage where CountryCode="JPN";
+-------------+----------------------+------------+------------+
| CountryCode | Language             | IsOfficial | Percentage |
+-------------+----------------------+------------+------------+
| JPN         | Ainu                 | F          |        0.0 |
| JPN         | Chinese              | F          |        0.2 |
| JPN         | English              | F          |        0.1 |
| JPN         | Japanese             | T          |       99.1 |
| JPN         | Korean               | F          |        0.5 |
| JPN         | MofuMofu             | T          |       10.0 |
| JPN         | Philippene Languages | F          |        0.1 |
+-------------+----------------------+------------+------------+
7 rows in set (0.00 sec)

@slave01
mysql> select * from countrylanguage where CountryCode="JPN";
+-------------+----------------------+------------+------------+
| CountryCode | Language             | IsOfficial | Percentage |
+-------------+----------------------+------------+------------+
| JPN         | Ainu                 | F          |        0.0 |
| JPN         | Chinese              | F          |        0.2 |
| JPN         | English              | F          |        0.1 |
| JPN         | Japanese             | T          |       99.1 |
| JPN         | Korean               | F          |        0.5 |
| JPN         | MofuMofu             | T          |       10.0 |
| JPN         | Philippene Languages | F          |        0.1 |
+-------------+----------------------+------------+------------+
7 rows in set (0.00 sec)
```

## 次は除外設定を入れてみる

Slave側に入れるパラメーターは
```
# my.cnf
replicate-ignore-table=world.countrylanguage

# SQL
change replication filter replicate_ignore_table = (world.countrylanguage);
```

```
@slave01
mysql> use mysql;
mysql> stop slave;
Query OK, 0 rows affected (0.01 sec)

mysql> change replication filter replicate_ignore_table = (world.countrylanguage);
Query OK, 0 rows affected (0.00 sec)

mysql> show slave status \G;
*************************** 1. row ***************************
               Slave_IO_State:
                  Master_Host: mysql-training-master01
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000005
          Read_Master_Log_Pos: 777
               Relay_Log_File: mysql-relay-bin.000002
                Relay_Log_Pos: 950
        Relay_Master_Log_File: mysql-bin.000005
             Slave_IO_Running: No
            Slave_SQL_Running: No
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table: world.countrylanguage
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 777
              Relay_Log_Space: 1157
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 1
                  Master_UUID: 41fcd501-ff66-11e8-b716-0242ac130003
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State:
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: 41fcd501-ff66-11e8-b716-0242ac130003:25-26
            Executed_Gtid_Set: 41fcd501-ff66-11e8-b716-0242ac130003:1-26
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)

mysql> start slave;
Query OK, 0 rows affected (0.00 sec)
```

# Ignoreされた事のテスト

```
@slave01
mysql> use world;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed

mysql> select * from countrylanguage where CountryCode="JPN";
+-------------+----------------------+------------+------------+
| CountryCode | Language             | IsOfficial | Percentage |
+-------------+----------------------+------------+------------+
| JPN         | Ainu                 | F          |        0.0 |
| JPN         | Chinese              | F          |        0.2 |
| JPN         | English              | F          |        0.1 |
| JPN         | Japanese             | T          |       99.1 |
| JPN         | Korean               | F          |        0.5 |
| JPN         | MofuMofu             | T          |       10.0 |
| JPN         | Philippene Languages | F          |        0.1 |
+-------------+----------------------+------------+------------+
7 rows in set (0.00 sec)

@master01
mysql>  insert into countrylanguage values ('JPN',"MofuMofu2","T","9.99");
Query OK, 1 row affected (0.02 sec)

mysql> select * from countrylanguage where CountryCode="JPN";
+-------------+----------------------+------------+------------+
| CountryCode | Language             | IsOfficial | Percentage |
+-------------+----------------------+------------+------------+
| JPN         | Ainu                 | F          |        0.0 |
| JPN         | Chinese              | F          |        0.2 |
| JPN         | English              | F          |        0.1 |
| JPN         | Japanese             | T          |       99.1 |
| JPN         | Korean               | F          |        0.5 |
| JPN         | MofuMofu             | T          |       10.0 |
| JPN         | MofuMofu2            | T          |       10.0 |
| JPN         | Philippene Languages | F          |        0.1 |
+-------------+----------------------+------------+------------+
8 rows in set (0.01 sec)

mysql> show master status;
+------------------+----------+--------------+------------------+-------------------------------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                         |
+------------------+----------+--------------+------------------+-------------------------------------------+
| mysql-bin.000005 |     1070 |              |                  | 41fcd501-ff66-11e8-b716-0242ac130003:1-27 |
+------------------+----------+--------------+------------------+-------------------------------------------+
1 row in set (0.00 sec)

@master01
mysql> select * from countrylanguage where CountryCode="JPN";
+-------------+----------------------+------------+------------+
| CountryCode | Language             | IsOfficial | Percentage |
+-------------+----------------------+------------+------------+
| JPN         | Ainu                 | F          |        0.0 |
| JPN         | Chinese              | F          |        0.2 |
| JPN         | English              | F          |        0.1 |
| JPN         | Japanese             | T          |       99.1 |
| JPN         | Korean               | F          |        0.5 |
| JPN         | MofuMofu             | T          |       10.0 |
| JPN         | Philippene Languages | F          |        0.1 |
+-------------+----------------------+------------+------------+
7 rows in set (0.00 sec)

mysql> show slave status \G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: mysql-training-master01
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000005
          Read_Master_Log_Pos: 1070
               Relay_Log_File: mysql-relay-bin.000003
                Relay_Log_Pos: 747
        Relay_Master_Log_File: mysql-bin.000005
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table: world.countrylanguage
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1070
              Relay_Log_Space: 1750
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 1
                  Master_UUID: 41fcd501-ff66-11e8-b716-0242ac130003
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: 41fcd501-ff66-11e8-b716-0242ac130003:25-27
            Executed_Gtid_Set: 41fcd501-ff66-11e8-b716-0242ac130003:1-27
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)

```

Slaveは更新されず、でも `show slave status` の `Executed_Gtid_Set` と `Master_Log_File` と `Read_Master_Log_Pos` が Masterと一致している事が確認できた。

## Slave対象外テーブルは消してみる
```
@slave01
mysql> truncate countrylanguage;
Query OK, 0 rows affected (0.01 sec)

mysql> select * from countrylanguage where CountryCode="JPN";
Empty set (0.00 sec)

@master01
mysql>  insert into countrylanguage values ('JPN',"MofuMofu3","T","9.99");
Query OK, 1 row affected (0.00 sec)

mysql> select * from countrylanguage where CountryCode="JPN";
+-------------+----------------------+------------+------------+
| CountryCode | Language             | IsOfficial | Percentage |
+-------------+----------------------+------------+------------+
| JPN         | Ainu                 | F          |        0.0 |
| JPN         | Chinese              | F          |        0.2 |
| JPN         | English              | F          |        0.1 |
| JPN         | Japanese             | T          |       99.1 |
| JPN         | Korean               | F          |        0.5 |
| JPN         | MofuMofu             | T          |       10.0 |
| JPN         | MofuMofu2            | T          |       10.0 |
| JPN         | MofuMofu3            | T          |       10.0 |
| JPN         | Philippene Languages | F          |        0.1 |
+-------------+----------------------+------------+------------+
9 rows in set (0.00 sec)

@slave01
mysql> select * from countrylanguage where CountryCode="JPN";
Empty set (0.00 sec)

※ 特にトレースにエラーも出ていない。

@master01
mysql> use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------------------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                         |
+------------------+----------+--------------+------------------+-------------------------------------------+
| mysql-bin.000005 |     1363 |              |                  | 41fcd501-ff66-11e8-b716-0242ac130003:1-28 |
+------------------+----------+--------------+------------------+-------------------------------------------+
1 row in set (0.00 sec)

@slave01

mysql> show slave status \G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: mysql-training-master01
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000005
          Read_Master_Log_Pos: 1363
               Relay_Log_File: mysql-relay-bin.000003
                Relay_Log_Pos: 1040
        Relay_Master_Log_File: mysql-bin.000005
             Slave_IO_Running: Yes

                  ~~~

           Replicate_Do_Table:
       Replicate_Ignore_Table: world.countrylanguage
      Replicate_Wild_Do_Table:

                  ~~~

           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: 41fcd501-ff66-11e8-b716-0242ac130003:25-28
            Executed_Gtid_Set: 41f66a7d-ff66-11e8-bda0-0242ac130002:1,
41fcd501-ff66-11e8-b716-0242ac130003:1-28
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)
```

Step1 おわり
