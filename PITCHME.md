@snap[text-09]
## MySQL数据库锁专题
@snapend
@snap[text-09]
@snapend



---?color=linear-gradient(90deg, white 50%, black 50%)
@snap[west span-40 text-center]

@ul[text-07 mt]
- 内容：锁的分类，死锁，元数据锁
- 实战：锁查看，锁分析，插入意向锁死锁，并发删除时造成死锁
- 要求：最好带笔记本电脑，安装mysql客户端工具
- 学习目标：了解mysql锁的基础知识，知道怎么去看锁的状态，查询死锁的原因和解决死锁。
@ulend

@snapend


@snap[north-east span-30 text-08]
@box[bg-gold](第一部分: Innodb锁分类)
@snapend

@snap[east span-30 text-08]
@box[bg-green](第二部分: Innodb中的死锁)
@snapend

@snap[south-east span-30 text-08]
@box[bg-blue](第三部分: MySQL中的元数据锁)
@snapend

---

### 实战1：上课环境准备


@snap[border-dashed-black]


```sh

#有ssh工具的人 ssh登录 
用户名：zzs2019 
密码：123456


# 没有ssh工具的人 登录网页 http://192.168.8.206:19822/zzs2019
用户名：zzs
密码：zzs2019 

点击宝塔终端开始

#练习 创建自己的mysql 
#注意修改  目录名称 zzs 映射端口 3307
mkdir -p ~/zzs/mysql/{data,logs}

docker run -p 3309:3306 --name mysql8_zzs -v $PWD/data:/var/lib/mysql \
 -v $PWD/logs:/logs -e MYSQL_ROOT_PASSWORD=123456 -d my_mysql8:8.1

2、/bin/bash 进入 bash 命令模式
$ docker exec -it test-mysql bash
```
+++ 
```sh

mysql -h127.0.0.1 -P3309 -uroot -p123456

use mysql;

update user set host = '192.168.*' where user = 'root' and host = '%' ;

update user set host = '%' where user = 'root' and host = 'localhost'  ;


ALTER USER 'root'@'%' IDENTIFIED BY '123456' PASSWORD EXPIRE NEVER;

ALTER USER 'root'@'192.168.*' IDENTIFIED BY '123456' PASSWORD EXPIRE NEVER;


ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';


ALTER USER 'root'@'192.168.*' IDENTIFIED WITH mysql_native_password BY '123456';

flush privileges;

```
+++ 
```sh

[root@zzs-lenovo-ll mysql]# mysql -h127.0.0.1 -P3309 -uroot -p123456
Enter password: 

MySQL [(none)]> exit
Bye
```
@snapend

---
##  行锁  

@snap[text-06 border-dashed-black]
@ul
- Shared and Exclusive Locks
- 行共享锁(S)与排他锁(X)较好理解， S锁与X锁互相冲突 
- 当读取当一行记录时为了防止别人修改则需要添加S锁 
- 当修改一行记录时为了防止别人同时进行修改则需要添加X锁 
- 这里需要知道MySQL中具有MVCC特性，所以通常情况下普通的查询属于一致性非锁定读不会添加任何锁，另外一种是锁定读例如 
- SELECT … FOR SHARE 
- 添加S共享锁，其它事务可以读但修改会被阻塞 
- SELECT … FOR UPDATE 
- 添加X排他锁，其它事务修改或者执行SELECT … FOR SHARE都会被阻塞 
@ulend
@snapend


+++

@snap[text-06 border-dashed-black]
@ul
- Record Locks-记录锁 
- MySQL中记录锁都是添加在索引上，即使表上没有索引也会在隐藏的聚集索引上添 加记录锁
- Gap Locks-间隙锁 
- 间隙锁锁定范围是索引记录之间的间隙或者第一个或最后一个索引记录之前的间隙，
- 例如一个事务执行:select * from t where c1 > 10 and c1 < 20 for update ; 那么当插入c1 
- 等于15时就会被阻塞否则再次查询得到结果就与第一次不一致 
- Next-Key Locks  
- Next-Key Locks是Record Locks与Gap Locks间隙锁的组合，也就是索引记录本身加上 
- 之前的间隙。间隙锁防止了保证RR级别下不出现幻读现象会，防止同一个事务内得
- 到的结果不一致。间隙锁在show engine innodb 输出如下 
@ulend
@snapend

+++

@snap[text-06 border-dashed-black]
@ul
- Insert Inten]on Locks-插入意向锁
- 插入意向锁定是在行插入之前由INSERT操作设置的一种间隙锁。 这个锁表示插入的意图，即插入相同索引间隙的多个事务如果不插入间隙内的相同位置则不需要等待彼此。 假设存在值为4和7的索引记录。分别尝试插入值5和6的事务分别在获取插入行上的排它锁之前用插入意图锁定锁定4和7之间的间隙， 但是不要互相阻塞，因为这些行是不冲突的。
@ulend
@snapend

+++

### Innodb中锁种类解读

@snap[text-06 border-dashed-black]
@ul
- 我们可以查看到锁情况，但是锁种类很多如何怎样判别？ 
- 从show innodb status 、 show engine innodb status\G 中看到的LOCK_MODE含义如下: 
- IX //意向锁拍他锁 
- X //Next-Key Lock 锁定记录本身和记录之前的间隙(X) 
- S //Next-Key Lock 锁定记录本身和记录之前的间隙(S) 
- X,REC_NOT_GAP// 代表只锁定记录本身(X)  
- S,REC_NOT_GAP // 代表只锁定记录本身(S) 
- Next-Key Locks是Record Locks与Gap Locks间隙锁的组合，也就是索引记录本身加上 
- X,INSERT_INTENTION //代表插入意向锁 
- X,GAP //代表GAP
@ulend
@snapend

---


## 表锁 

@snap[text-06 border-dashed-black]
@ul
-  Intention Locks-意向锁:意向锁在MySQL中是表级别锁，表示将来要在表上什么类型的锁(IX/IS) 
- SELECT … FOR SHARE 意向共享锁(IS):要获取表中某行的共享锁之前，它必须首先在表 上获取IS锁 
- SELECT … FOR UPDATE 意向排他锁(IX):要获取表中某行的独占锁之前，它必须首先获取表上的IX锁 
- AUTO-INC Locks-自增锁 
- AUTO-INC锁是由插入到具有AUTO_INCREMENT列的表中的事务所采用的特殊表级锁。在最简单的情况下，如果一个事务正在向表中插入值，则任何其他事务必须等待对该表执行自己的插入，以便第一个事务插入的行接收连续的主键值。  
- 参数innodb_autoinc_lock_mode控制自增锁的算法，可以控制自增值生成的策略来提高并发
@ulend
@snapend

---

### Innodb中加锁情况分析 
#### 实战2： 建测试表，分析各种隔离级别的加锁情况
@snap[text-06 border-dashed-black]
@ul
- 不同的隔离级别及条件字段是否为主键/索引相关 
- CREATE TABLE `t` (`id` int(11) DEFAULT NULL,`name` char(20) DEFAULT NULL,
- PRIMARY KEY (`id`) USING );
- insert into t values (10,'zzs'),(20,'zzy'),(30,'zjc'); 
- CREATE TABLE `t2` (`id` int(11) DEFAULT NULL,`name` char(20) DEFAULT NULL);
- insert into t values (10,'zzs'),(20,'zzy'),(30,'zjc'); 
@ulend
@snapend



+++

##### 事务隔离级别为RR表中无显式主键与索引: 

- select * from t2 for update; 
- select * from t2 where id = 10 for update; 

@snap[border-dashed-black]
```sql
MySQL [db_test]> begin;
Query OK, 0 rows affected (0.00 sec)

MySQL [db_test]> select * from t2 for update;
+------+------+
| id   | name |
+------+------+
|   10 | zzs  |
|   20 | zzy  |
|   30 | zjc  |
+------+------+
3 rows in set (0.00 sec)


```


+++

@snap[text-04 border-dashed-black]
```sql
MySQL [performance_schema]> SELECT t.ENGINE_LOCK_ID, t.ENGINE_TRANSACTION_ID, t.THREAD_ID, t.EVENT_ID, 
t.OBJECT_SCHEMA, t.OBJECT_NAME, t.INDEX_NAME, t.LOCK_TYPE, t.LOCK_MODE, t.LOCK_STATUS,t.LOCK_DATA FROM data_locks AS t \G
*************************** 1. row ***************************
       ENGINE_LOCK_ID: 140285350388848:1065:140285241799576
ENGINE_TRANSACTION_ID: 2631
            THREAD_ID: 61
             EVENT_ID: 25
        OBJECT_SCHEMA: db_test
          OBJECT_NAME: t2
           INDEX_NAME: NULL
            LOCK_TYPE: TABLE
            LOCK_MODE: IX
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
       ENGINE_LOCK_ID: 140285350388848:4:4:1:140285241796696
ENGINE_TRANSACTION_ID: 2631
            THREAD_ID: 61
             EVENT_ID: 25
        OBJECT_SCHEMA: db_test
          OBJECT_NAME: t2
           INDEX_NAME: GEN_CLUST_INDEX
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: supremum pseudo-record
*************************** 3. row ***************************
       ENGINE_LOCK_ID: 140285350388848:4:4:2:140285241796696
ENGINE_TRANSACTION_ID: 2631
            THREAD_ID: 61
             EVENT_ID: 25
        OBJECT_SCHEMA: db_test
          OBJECT_NAME: t2
           INDEX_NAME: GEN_CLUST_INDEX
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: 0x000000000200
*************************** 4. row ***************************
       ENGINE_LOCK_ID: 140285350388848:4:4:3:140285241796696
ENGINE_TRANSACTION_ID: 2631
            THREAD_ID: 61
             EVENT_ID: 25
        OBJECT_SCHEMA: db_test
          OBJECT_NAME: t2
           INDEX_NAME: GEN_CLUST_INDEX
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: 0x000000000201
*************************** 5. row ***************************
       ENGINE_LOCK_ID: 140285350388848:4:4:4:140285241796696
ENGINE_TRANSACTION_ID: 2631
            THREAD_ID: 61
             EVENT_ID: 25
        OBJECT_SCHEMA: db_test
          OBJECT_NAME: t2
           INDEX_NAME: GEN_CLUST_INDEX
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: 0x000000000202
5 rows in set (0.00 sec)

```
@snapend

+++
@snap[text-05 border-dashed-black]
- 切换RR隔离级别
- show variables like '%transaction_isolation%';
- set GLOBAL transaction_isolation='REPEATABLE-READ';
- RC隔离级别+表无显式主键和索引: 
- RR隔离级别+有显式主键无索引: 
- select * from t for update;
- select * from t where id = 10 for update;
- RR隔离级别+有显式主键无索引: 
- select * from t where id = 10 and name = 'zzs' for update;

@snapend

+++

```sql
CREATE TABLE `t3` (
`id` int(11) DEFAULT NULL,
`name` char(20) DEFAULT NULL
);

insert into t3 values (10,'zzs'),(20,'zzy'),(30,'zjc');

create index idx_id on t3 (id);
```

@snap[text-05 border-dashed-black]
- RR隔离级别+无显式主键有索引: 
- 普通索引 
- select * from t3 where id = 10 for update; 
- insert into t3 values (9, 'zzy1');
- RR隔离级别+无显式主键有索引: 
- 普通索引 
- insert into t3 values (19, 'zzu2'); 
- insert into t3 values (20, 'zzu2');
- RR隔离级别+无显式主键有索引: 
- 唯一索引 
- select * from t where id = 10 for update;
- RR隔离级别+有显式主键有索引: 
- 有显示主键普通索引 
- select * from t for update;

@snapend

+++
@snap[text-05 border-dashed-black]
- 切换RC隔离级别
- show variables like '%transaction_isolation%';
- set GLOBAL transaction_isolation='READ-COMMITTED';
- select * from t for update; 
- select * from t where id = 10 for update; 
- RC隔离级别+无显式主键有索引: 
- 普通索引 
- select * from t where id = 10 for update; 
- RC隔离级别+有显式键有索引: 
- 不带where条件 
- select * from t for update; 

@snapend

---
#### 第二部分: Innodb中的死锁
##### 死锁的产生 
@snap[text-06 border-dashed-black]

- 当两个事务都试图获取另一个事务已经拥有的锁时，就会发生死锁 
- 但会有一些不经意的地方会产生死锁
- 现象 实验1
@snapend
+++

```sql
+--------------+--------------+
|      session1|      session2|
+--------------+--------------+
update t set name = 'kkk1' where id = 20;|空|
|空|select * from t lock in share mode; 发生柱塞 |
|insert into t valuses(11,'zzz111');|死锁|
MySQL [db_test]> set autocommit=0;
Query OK, 0 rows affected (0.00 sec)

MySQL [db_test]> select @@autocommit;
+--------------+
| @@autocommit |
+--------------+
|            0 |
+--------------+
1 row in set (0.00 sec)
session1
MySQL [db_test]> begin;
Query OK, 0 rows affected (0.00 sec)
MySQL [db_test]> update t set name = 'kkk1' where id = 20;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0
MySQL [db_test]> insert into t valuses(11,'zzz111');
ERROR 1064 (42000): You have an error in your SQL syntax; check the 
manual that corresponds to your MySQL server version for the right 
syntax to use near 'valuses(11,'zzz111')' at line 1

```
+++
```sql
session2
MySQL [db_test]> set autocommit=0;
Query OK, 0 rows affected (0.00 sec)

MySQL [db_test]> select @@autocommit;
+--------------+
| @@autocommit |
+--------------+
|            0 |
+--------------+
1 row in set (0.00 sec)

MySQL [db_test]> begin;
Query OK, 0 rows affected (0.00 sec)

MySQL [db_test]> select * from t lock in share mode;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

```


+++
```sql
MySQL [performance_schema]> SELECT t.ENGINE_LOCK_ID, t.ENGINE_TRANSACTION_ID, t.THREAD_ID, t.EVENT_ID, t.OBJECT_SCHEMA, t.OBJECT_NAME, t.INDEX_NAME, t.LOCK_TYPE, t.LOCK_MODE, t.LOCK_STATUS,t.LOCK_DATA FROM data_locks AS t ;
+---------------------------------------+-----------------------+-----------+----------+---------------+-------------+------------+-----------+---------------+-------------+-----------+
| ENGINE_LOCK_ID                        | ENGINE_TRANSACTION_ID | THREAD_ID | EVENT_ID | OBJECT_SCHEMA | OBJECT_NAME | INDEX_NAME | LOCK_TYPE | LOCK_MODE     | LOCK_STATUS | LOCK_DATA |
+---------------------------------------+-----------------------+-----------+----------+---------------+-------------+------------+-----------+---------------+-------------+-----------+
| 140285350388848:1063:140285241799576  |                  2669 |        61 |       56 | db_test       | t           | NULL       | TABLE     | IX            | GRANTED     | NULL      |
| 140285350388848:2:4:3:140285241796696 |                  2669 |        61 |       56 | db_test       | t           | PRIMARY    | RECORD    | X,REC_NOT_GAP | GRANTED     | 20        |
| 140285350391440:1063:140285241817448  |       421760327102096 |        65 |       33 | db_test       | t           | NULL       | TABLE     | IS            | GRANTED     | NULL      |
| 140285350391440:2:4:2:140285241814664 |       421760327102096 |        65 |       33 | db_test       | t           | PRIMARY    | RECORD    | S             | GRANTED     | 10        |
| 140285350391440:2:4:3:140285241815008 |       421760327102096 |        65 |       33 | db_test       | t           | PRIMARY    | RECORD    | S             | WAITING     | 20        |
+---------------------------------------+-----------------------+-----------+----------+---------------+-------------+------------+-----------+---------------+-------------+-----------+
5 rows in set (0.00 sec)


```
+++

##### 实战:插入意向锁死锁 

@snap[text-06 border-dashed-black]
- Next-Key Lock锁与插入意向锁兼容情况
- session2等待session1上X锁的释放，随后的插入意向锁与session2 
- GAP S-lock(Next-Key Lock)不兼容， 这样就会造成session1与 
- session2都不能同时进行下去了造成了死锁
- 出现死锁时选择回滚哪个事务？ 
- Innodb中出现死锁时选择回滚代价小的事务 
- 通过innodb_trx表中的trx_weight来判断占用资源的大小，此案例中单独去 
- 执行SQL通过查询innodb_trx表分别对应的trx_weight是 
- 语句 trx_weight 
@snapend

+++
##### 实战:并发删除时造成死锁 
- 如何解读死锁日志？ 
- 两个事务信息 
- 事务xxxxx在执行delete语句是发生了锁等待
---

### 第三部分: MySQL中的元数据锁

@snap[text-06 border-dashed-black]
- MySQL中还具有一种表级别锁 MDL锁，防止读写能正常 
- 当对一个表做增删改查操作的时候，加 MDL 读锁 
- 读锁之间不互斥，可以多个线程同时对一张表增删改查 
- 当要对表做结构变更操作的时候，加 MDL 写锁 
- 读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的安全性。如果有两个 
- 线程要同时给一个表加字段，其中一个要等另一个执行完才能开始执行 
- 线上对表做DDL操作时需要避免有大事务存在 
- 会导导致业务不能正常访问

@snapend

+++

@snap[text-06 border-dashed-black]
- MySQL中元数据锁与备份之间关系 
- Xtrbackup备份当中会对元数据申请MDL锁，所以备份时如果有长时间未执 
- 行完的SQL语句会导致备份失败 
- mysqldump备份中也会存在相似情况，但有些不同 
- 如果ddl语句执行时mysqldump正处于select 数据阶段则ddl语句会被阻塞住不能执行 
- 如果ddl语句执行时mysqldump处于刚show create table和select 数据之间，则ddl语句 
- 能正常执行，但是mysqldump后面执行时就会报错终止 
- Table defifini]on has changed, please retry transac]on 
@snapend