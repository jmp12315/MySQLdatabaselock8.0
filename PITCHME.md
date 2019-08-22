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

##  行锁

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

##  行锁

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