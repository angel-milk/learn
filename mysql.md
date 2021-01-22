#### mysql 事务
* 特性
1. 原子性 事务是数据库的逻辑工作单位，事务中包括的操作要不全做，要么都不做。
2. 一致性 事务的执行结果必须是使数据库从一个一致性的状态转变为另一个一致性的状态，事务执行不成功，已经执行的操作需要回滚。
3. 隔离线 一个事务的执行不被其他事务干扰。
4. 持续性 事务提交成功，对数据库的更改是永久的。
* 隔离级别
1. 读未提交 READ-UNCOMMITED 0 存在脏读，不可重复读，幻读
2. 读已提交 READ-COMMITED   1 解决脏读，存在不可重复读，幻读
3. 可重复读 REPEATABLE-READ 2 mysql默认级别，解决脏读，不可重复读问题，存在幻读问题，使用mvcc机制可以实现可重复读
4. 序列号   SERIALIZABLE    3 解决脏读，不可重复读，幻读，可保证事务安全，但是串行执行，性能最低。

#### 隔离级别
```mysql
select @@global.tx_isolation, @@tx_isolation;
set @@global.tx_isolation = 0
set @@global.tx_isolation = 'READ-UNCOMMITED'
```

#### 幻读（存疑）
##### 事务1
1. select * from user where id = 1; // empty set 
2. begin
##### 事务2
3. begin
4. insert into user values(1, 'test');
5. commit
6. select * from user where id = 1; // 1, test
##### 事务1 
7. insert into user values(1, 'test'); //duplicate entry 1 from key primary (发生幻读)
8. select * from user where id = 1; // empty set

#### 避免RR级别下幻读
1. 对select手动加行（X）锁
> * select id from user where id = 1 for update
> * id = 1 时记录会被加行锁。不存在加next-lock/gap（范围行锁）锁。即记录存在与否，mysql都会对相应索引加锁，其他事务无法再次获得操作。 

##### 事务1
1. select * from user where id = 1 for update; // empty set 加行（X）锁 
2. begin
##### 事务2
3. begin
4. insert into user values(1, 'test'); // 被T1阻塞接步骤7
##### 事务1 
5. insert into user values(1, 'test'); //query ok
6. commit // 提交唤醒T2执行
##### 事务2 
7. 执行，返回duplicate entry 1 from key primary


#### mvcc 基于乐观锁实现的隔离级别方式
> innodb mvcc，通过每行记录后保存两个隐藏列实现的。一行保存行创建时间，一行保存运营过期时间(系统版本号)。每开始一个新事务，系统版本号就会自动递增。事务开始时刻，系统版本号作为事务的版本号，用于查询来的记录进行版本号比较。
* select : 
> 1. innodb只查找版本早于当前事务版本的数据行，行的系统版本号小于或者等于事务的系统版本号。可以确保事务读取的行，要么事务开始已经存在，要不事务自身插入或者修改。
> 2. 行的删除版本要么未定义，要么当于当前事务版本号，确保事务读的行，在事务开始之前。
* insert :
> innodb为新插入的每一行保存当前系统版本号为行版本号。
* delete :
> innodb为删除的每一行保存当前系统版本号为行删除标识。
* update :
> innodb为插入一行新记录，保存当前系统版本号为行版本号，同时保存当前系统版本号为原来行行为为删除行为。

##### 优缺点
> 保存这两个额外的版本号，大多数操作不用加锁。使得数据库操作简单，性能很好，保证了只会读取到符合标准的行。不足是需要额外的存储空间，一些维护开销。
* mvcc 只在REATABLE-READ,READ-COMMITED级别下工作。

#### mvcc 可重复读
* session A
```sql
begin;
select * from zgz_test;
```
```
id name
1  test1
2  t2
```
* session B
```sql
begin;
select * from zgz_test;
update zgz_test set name = 't3' where id = 2;
commit;
```
* session A
```sql
select * from zgz_test;
```
```
id name
1  test1
2  t2
```
* session A
```sql
commit
```

####总结
> * 版本未提交，不可见。
> * 版本已提交，但是在快照创建后提交，不可见。
> * 版本已提交，而且在快照创建前提交，可见。


### 当前读，快照读
* 快照读(select默认)
> 读取记录数据的可见版本（可能是过期数据），不加锁
* 当前读(update,insert,delete默认)
> 读取记录数据的最新版本，并且当前读返回的记录都会加上锁，保证其他事务不会并发修改这条记录。

### 脏读
> 脏读：无效数据被读取出来。一个事务读取另外一个事务未提交的数据。
* 解决办法： 事务等级调整READ_COMMITED

### 不可重复读
> 不可重复读：同时操作，事务分别读取事务二操作时和提交后的数据，读取数据不一致。（在同一个事务中，两次相同查询返回了不同的结果。）
* 解决办法： 事务等级调整REPEATABLE_READ

### 幻读
> 幻读：幻读是事务非独立执行时发生的一种现象。例如事务T1对一个表中所有的行的某个数据项做了从“1”修改为“2”的操作，这时事务T2又对这个表中插入了一行数据项，而这个数据项的数值还是为“1”并且提交给数据库。而操作事务T1的用户如果再查看刚刚修改的数据，会发现还有一行没有修改，其实这行是从事务T2中添加的，就好像产生幻觉一样，这就是发生了幻读。幻读和不可重复读都是读取了另一条已经提交的事务(这点就脏读不同)，所不同的是不可重复读可能发生在update,delete操作中，而幻读发生在insert操作中。


### 排他锁，共享锁
* 排它锁（exclusive）：x锁，写锁
* 共享锁（shared）：s锁，读锁


### mvcc 幻读
> READ-REPEATABLE ,INNODB使用mvcc和next-key locks解决幻读。mvcc解决普通快照读的幻读，next-key locks解决当前读的幻读

### 锁
* 行锁（record lock） 锁直接加到索引记录上，某一行
* 间隙锁（gap lock）锁加到空隙中，不在行上。
* next-key lock 行锁和间隙锁组合起来
> Next-key锁是在下一个索引记录本身和索引之前的gap加上S锁或是X锁(如果是读就加上S锁，如果是写就加X锁)。

#### for update， lock in share mode
* for update (意向排他锁)
* lock in share mode （意向共享锁） 
1. lock in share mode 所有符合条件的rows都会加上共享锁，其他session都会读取这些记录，也可继续添加is锁，但无法修改激励，直至释放is锁。
2. for update 所有符合条件都加上排它锁，其他session无法在添加任务s和x锁。但非is锁和ix锁session快照读不阻塞。

#### 相同点：
> 1.两者都会对并发的操作造成阻塞，等待A操作完成；
> 2.查询操作不会造成阻塞(不带for update)
> 3.操作阻塞（带for update）
 

#### 不同点：
> 1. 并发时for update会使B一直阻塞，等待A操作完成后执行B操作；
> 2. 而在使用lock in share mode下当B阻塞时，如果A继续有修改数据，那么此时B会终止失败
