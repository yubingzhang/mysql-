## InnoDB锁的类型 ##

**1.InnoDB引擎使用了七种类型的锁，他们分别是：**  

•	共享排他锁（*Shared and Exclusive Locks*）  
•	意向锁（*Intention Locks*）  
•	记录锁（*Record Locks*）  
•	间隙锁（*Gap Locks*）  
•	Next-Key Locks  
•	插入意图锁（*Insert Intention Locks*）  
•   自增锁（*AUTO-INC Locks*）  



**1.1 Shared and Exclusive Locks**  

共享锁（S锁）和排他锁（X锁）的概念在许多编程语言中都出现过。先来描述一下这两种锁在MySQL中的影响结果：  
•	如果一个事务对某一行数据加了S锁，另一个事务还可以对相应的行加S锁，但是不能对相应的行加X锁。  
•	如果一个事务对某一行数据加了X锁，另一个事务既不能对相应的行加S锁也不能加X锁。


**1.2 Record Locks、Gap Locks、Next-Key Locks**


•	记录锁（Record Locks）:记录锁锁定索引中一条记录。  
•	间隙锁（Gap Locks）:间隙锁要么锁住索引记录中间的值，要么锁住第一个索引记录前面的值或者最后一个索引记录后面的值。  
•	Next-Key Locks:Next-Key锁是索引记录上的记录锁和在索引记录之前的间隙锁的组合。
定义中都提到了索引记录（index record）。为什么？行锁和索引有什么关系呢？其实，InnoDB是通过搜索或者扫描表中索引来完成加锁操作，InnoDB会为他遇到的每一个索引数据加上共享锁或排他锁。所以我们可以称行级锁（row-level locks）为索引记录锁（index-record locks），因为行级锁是添加到行对应的索引上的。  

三种类型锁的锁定范围不同，且逐渐扩大。我们来举一个例子来简要说明各种锁的锁定范围，假设表t中索引列有3、5、8、9四个数字值，根据官方文档的确定三种锁的锁定范围如下：  
•	记录锁的锁定范围是单独的索引记录，就是3、5、8、9这四行数据。  
•	间隙锁的锁定为行中间隙，用集合表示为(-∞,3)、(3,5)、(5,8)、(8,9)、(9,+∞)。  
•	Next-Key锁是有索引记录锁加上索引记录锁之前的间隙锁组合而成，用集合的方式表示为(-∞,3]、(3,5]、(5,8]、(8,9]、(9,+∞)。  
***最后对于间隙锁还需要补充三点：***    
1.	间隙锁阻止其他事务对间隙数据的并发插入，这样可有有效的解决幻读问题(Phantom Problem)。正因为如此，并不是所有事务隔离级别都使用间隙锁，MySQL InnoDB引擎只有在Repeatable Read（默认）隔离级别才使用间隙锁。  
2.	间隙锁的作用只是用来阻止其他事务在间隙中插入数据，他不会阻止其他事务拥有同样的的间隙锁。这就意味着，除了insert语句，允许其他SQL语句可以对同样的行加间隙锁而不会被阻塞。  
3.	对于唯一索引的加锁行为，间隙锁就会失效，此时只有记录锁起作用。





**2. 加锁语句**  

前面我们已经介绍了InnoDB的是在SQL语句的执行过程中通过扫描索引记录的方式来实现加锁行为的。那哪些些语句会加锁？加什么样的锁？接下来我们逐一描述：  
**•**	select ... from语句：InnoDB引擎采用多版本并发控制（MVCC）的方式实现了非阻塞读，所以对于普通的select读语句，InnoDB并不会加锁【注1】。  
**•**	select ... from lock in share mode语句：这条语句和普通select语句的区别就是后面加了lock in share mode，通过字面意思我们可以猜到这是一条加锁的读语句，并且锁类型为共享锁（读锁）。InnoDB会对搜索的所有索引记录加next-key锁，但是如果扫描的唯一索引的唯一行，next-key降级为索引记录锁。  
**•**	select ... from for update语句：和上面的语句一样，这条语句加的是排他锁（写锁）。InnoDB会对搜索的所有索引记录加next-key锁，但是如果扫描唯一索引的唯一行，next-key降级为索引记录锁。  
**•**	update ... where ...语句：。InnoDB会对搜索的所有索引记录加next-key锁，但是如果扫描唯一索引的唯一行，next-key降级为索引记录锁。【注2】  
**•**	delete ... where ...语句：。InnoDB会对搜索的所有索引记录加next-key锁，但是如果扫描唯一索引的唯一行，next-key降级为索引记录锁。
**•**	insert语句：InnoDB只会在将要插入的那一行上设置一个排他的索引记录锁。  
**最后补充两点：**  
1.	如果一个查询使用了辅助索引并且在索引记录加上了排他锁，InnoDB会在相对应的聚合索引记录上加锁。  
2.	如果你的SQL语句无法使用索引，这样MySQL必须扫描整个表以处理该语句，导致的结果就是表的每一行都会被锁定，并且阻止其他用户对该表的所有插入。




**3.几种常见加锁场景**

**•** 辅助索引列上面添加X锁，其他事物添加数据时，如果添加的辅助索引列的值不是next-key锁边界值，则辅助索引的前后会形成next-key锁  

**•** 辅助索引列上面添加X锁，其他事物添加数据时，如果添加的辅助索引列的值是next-key锁边界值，	则有如下三种锁定行为：    
*1.	对SQL语句扫描过的辅助索引记录行加上next-key锁（注意也锁住记录行之后的间隙）。  
2.	对辅助索引对应的聚合索引加上索引记录锁。  
3.	当辅助索引为间隙锁“最小”和“最大”值时，对聚合索引相应的行加间隙锁。“最小”锁定对应聚合索引之后的行间隙。“最大”值锁定对应聚合索引之前的行间隙。*

**•** 唯一索引或主键索引加锁，间隙锁失效，只对单条记录加索引记录锁  

**•** <font color="red">执行计划是全表扫描，就算使用到索引了，也会对整个表加锁  </font>


**•** 结合SQL语句select * from user where name>'e' for update;执行后返回结果我们判断这两行记录应该为g和i。  
因为select * from user where name>'e' for update;语句扫描了两行索引记录分别是g和i，所以我们将g和i的锁定范围叠就可以得到where name>'e'的锁定范围：  

*1.	索引记录g在name列锁定范围为(e,g],(g,i)。索引记录i的在name列锁定范围为(g,i],(i,+∞)。两者叠加后锁定范围为(e,g],(g,i],(i,+∞)。其中g,i为索引记录锁。  
2.	g和i对应id列中的7和9加索引记录锁。  
3.	当name列的值为锁定范围上边界e时，还会在e所对应的id列值为5之后的所有值之间加上间隙锁，范围为(5,7),(7,9),(9,+∞)。下边界为+∞无需考虑。*
