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


@snap[north-east span-40 text-08]
@box[bg-green](第一部分: Innodb锁分类)
@snapend

@snap[east span-40 text-08]
@box[bg-blue](第二部分: Innodb中的死锁)
@snapend

@snap[south-east span-40 text-08]
@box[bg-gold](第三部分: MySQL中的元数据锁)
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
-  Inten]on Locks-意向锁:意向锁在MySQL中是表级别锁，表示将来要在表上什么类型的锁(IX/IS) 
- SELECT … FOR SHARE 意向共享锁(IS):要获取表中某行的共享锁之前，它必须首先在表 上获取IS锁 
- SELECT … FOR UPDATE 意向排他锁(IX):要获取表中某行的独占锁之前，它必须首先获取表上的IX锁 
- AUTO-INC Locks-自增锁 
- AUTO-INC锁是由插入到具有AUTO_INCREMENT列的表中的事务所采用的特殊表级锁。在最简单的情况下，如果一个事务正在向表中插入值，则任何其他事务必须等待对该表执行自己的插入，以便第一个事务插入的行接收连续的主键值。  
- 参数innodb_autoinc_lock_mode控制自增锁的算法，可以控制自增值生成的策略来提高并发
@ulend
@snapend