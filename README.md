## Database Scalability

### Indexing
### De-normalization

### DB Replication 
**read-only replicas**  1 master/N slaves case, all update goes to master DB which send **a change log** to the replicas. However, there 
will be a **time lag** for replication.

### Table Partitioning

You can partition vertically or horizontally.

Vertical partitioning is about putting different DB tables into different machines or moving some columns (rarely access attributes) to a different table.

### Transaction Processing

Avoid mixing OLAP (Online anylasis processing, query intensive) and OLTP(Online transaction processing, update intensive) operations within the same DB. 

In the OLTP system, avoid using long running database transaction and choose the isolation level appropriately. 
A typical technique is to use optimistic business transaction. 
Under this scheme, a long running business transaction is executed outside a database transaction. 



## Relational database management system (RDBMS)

**ACID** is a set of properties of relational database [transactions](https://en.wikipedia.org/wiki/Database_transaction).

* **Atomicity** - Each transaction is all or nothing
* **Consistency** - Any transaction will bring the database from one valid state to another
* **Isolation** - **Executing transactions concurrently** has the same results as if the transactions were executed **serially**
* **Durability** - Once a transaction has been committed, it will remain so

There are many techniques to scale a relational database: **master-slave replication**, **master-master replication**, **federation**, **sharding**, **denormalization**, and **SQL tuning**.

#### Master-slave replication

The master serves **reads and writes**, replicating writes to one or more **slaves, which serve only reads**.  
Slaves can also replicate to additional slaves in a tree-like fashion.  
If the master goes offline, the system can continue to operate in read-only mode until **a slave is promoted** to a master or a new master is provisioned.

<p align="center">
  <img src="http://i.imgur.com/C9ioGtn.png"/>
  <br/>
  <i><a href=http://www.slideshare.net/jboner/scalability-availability-stability-patterns/>Source: Scalability, availability, stability, patterns</a></i>
</p>

##### Disadvantage(s): master-slave replication

* Additional logic is needed to **promote a slave to a master**.
* See [Disadvantage(s): replication](#disadvantages-replication) for points related to **both** master-slave and master-master.



#### Master-master replication


<p align="center">
  <img src="http://i.imgur.com/krAHLGg.png"/>
  <br/>
  <i><a href=http://www.slideshare.net/jboner/scalability-availability-stability-patterns/>Source: Scalability, availability, stability, patterns</a></i>
</p>

##### Disadvantage(s): master-master replication

* need a **load balancer** 
* Most master-master systems are either **loosely consistent (violating ACID)** or have **increased write latency due to synchronization**.
* See [Disadvantage(s): replication](#disadvantages-replication) for points related to **both** master-slave and master-master.

##### Disadvantage(s): replication

* There is a potential for **loss of data if the master fails before any newly written data** can be replicated to other nodes.
* Writes are replayed to the read replicas.  If there are a lot of writes, the read replicas can get bogged down(阻塞) with replaying writes and can't do as many reads.
* The more read slaves, the more you have to replicate, which leads to greater replication lag.


##### Source(s) and further reading: replication

* [Scalability, availability, stability, patterns](http://www.slideshare.net/jboner/scalability-availability-stability-patterns/)
* [Multi-master replication](https://en.wikipedia.org/wiki/Multi-master_replication)



#### Federation

<p align="center">
  <img src="http://i.imgur.com/U3qV33e.png"/>
  <br/>
  <i><a href=https://www.youtube.com/watch?v=w95murBkYmU>Source: Scaling up to your first 10 million users</a></i>
</p>

Federation (or functional partitioning) **splits up databases by function**.  Smaller databases result in more data that can fit in memory, which in turn results in more cache hits due to improved cache locality.  With no single central master serializing writes you can write in parallel, increasing throughput.

##### Disadvantage(s): federation

* not effective if your schema requires huge functions or tables.
* application logic to determine which database to read and write.
* **Joining** data from two databases is more complex with a [server link](http://stackoverflow.com/questions/5145637/querying-data-by-joining-two-tables-in-two-database-on-different-servers).


#### Sharding

<p align="center">
  <img src="http://i.imgur.com/wU8x5Id.png"/>
  <br/>
  <i><a href=http://www.slideshare.net/jboner/scalability-availability-stability-patterns/>Source: Scalability, availability, stability, patterns</a></i>
</p>

Sharding distributes data **across different databases** such that each database can only manage a subset of the data.  


Similar to the advantages of [federation](#federation), sharding results in 
**less read and write traffic, less replication, and more cache hits.**  
Index size is also reduced.  
If one shard goes down, the other shards are still operational.  
allowing you to **write in parallel with increased throughput**.


##### Disadvantage(s): sharding

*application logic to work with shards, **complex SQL queries**.
* LOAD NOT BALANCED
    * Rebalancing adds additional complexity.  A sharding function based on [consistent hashing](http://www.paperplanes.de/2011/12/9/the-magic-of-consistent-hashing.html) can reduce the amount of transferred data.
* Joining data from multiple shards is more complex.


#### Denormalization

**Redundant copies of the data** are written in multiple tables to avoid expensive joins.  Some RDBMS such as [PostgreSQL](https://en.wikipedia.org/wiki/PostgreSQL) support [materialized views](https://en.wikipedia.org/wiki/Materialized_view) which handle  storing redundant information and keeping redundant copies consistent.


##### Disadvantage(s): denormalization

* Data is duplicated.
* redundant copies of information stay in sync, which increases complexity of the database design.
* A denormalized database under **heavy write load** might perform worse than its normalized counterpart.


#### SQL tuning

It's important to **benchmark** and **profile** to simulate and uncover bottlenecks.

* **Benchmark** - Simulate high-load situations with tools such as [ab](http://httpd.apache.org/docs/2.2/programs/ab.html).
* **Profile** - Enable tools such as the [slow query log](http://dev.mysql.com/doc/refman/5.7/en/slow-query-log.html) to help track performance issues.




##### Use good indices

* Columns that you are querying (`SELECT`, `GROUP BY`, `ORDER BY`, `JOIN`) could be faster with indices.
* Indices are usually represented as self-balancing [B-tree](https://en.wikipedia.org/wiki/B-tree) that keeps data sorted and allows searches, sequential access, insertions, and deletions in logarithmic time.
* Placing an **index can keep the data in memory**, requiring more space.
* Writes could also be slower since the index also needs to be updated.
* When loading large amounts of data, it might be faster to disable indices, load the data, then rebuild the indices.


##### Partition tables

* Break up a table by putting hot spots in a separate table to help keep it in memory.

##### Tune the query cache

* In some cases, the [query cache](https://dev.mysql.com/doc/refman/5.7/en/query-cache.html) could lead to [performance issues](https://www.percona.com/blog/2016/10/12/mysql-5-7-performance-tuning-immediately-after-installation/).

##### Source(s) and further reading: SQL tuning

* [Tips for optimizing MySQL queries](http://aiddroid.com/10-tips-optimizing-mysql-queries-dont-suck/)
* [Is there a good reason i see VARCHAR(255) used so often?](http://stackoverflow.com/questions/1217466/is-there-a-good-reason-i-see-varchar255-used-so-often-as-opposed-to-another-l)
* [How do null values affect performance?](http://stackoverflow.com/questions/1017239/how-do-null-values-affect-performance-in-a-database-search)
* [Slow query log](http://dev.mysql.com/doc/refman/5.7/en/slow-query-log.html)







# Join

**Nest Loop Join**算法虽然可以借助连接列索引，但是带来的随机读成本过大。

**Merge Sort Join**虽然可以减少随机读的情况，但是带来的大规模Sort操作，对内存和Temp空间压力过大。两种算法在处理海量数据的时候，如果是海量随机读还是海量排序，都是不能被接受的连接算法。本篇中，我们介绍目前比较常用的一种连接方式Hash Join连接。
nested loop join

　　嵌套循环连接，是比较通用的连接方式，分为内外表，每扫描外表的一行数据都要在内表中查找与之相匹配的行，没有索引的复杂度是O(N*M)，这样的复杂度对于大数据集是非常劣势的，一般来讲会通过索引来提升性能。　

 　　sort merge-join

　　merge join需要首先对两个表按照关联的字段进行排序，分别从两个表中取出一行数据进行匹配，如果合适放入结果集；不匹配将较小的那行丢掉继续匹配另一个表的下一行，依次处理直到将两表的数据取完。merge join的很大一部分开销花在排序上，也是同等条件下差于hash join的一个主要原因。
  
  
## Hash Join算法步骤：

  i.       Hash Join连接对象依然是两个数据表，首先选择出其中一个“小表”。这里的小表，就是参与连接操作的数据集合数据量小。对连接列字段的所有数据值，进行Hash函数操作。Hash函数是计算机科学中经常使用到的一种处理函数，利用Hash值的快速搜索算法已经被认为是成熟的检索手段。Hash函数处理过的数据特征是“相同数据值的Hash函数值一定相同，不同数据值的Hash函数值可能相同”；

 ii.       经过Hash处理过的小表连接列，连同数据一起存放到Oracle PGA空间中。PGA中存在一块空间为hash_area，专门存放此类数据。并且，依据不同的Hash函数值，进行划分Bucket操作。每个Bucket中包括所有相同hash函数值的小表数据。同时建立Hash键值对应位图。

iii.       之后对进行Hash连接大表数据连接列依次读取，并且将每个Hash值进行Bucket匹配，定位到适当的Bucket上（应用Hash检索算法）；

iv.       在定位到的Bucket中，进行小规模的精确匹配。因为此时的范围已经缩小，进行匹配的成功率精确度高。同时，匹配操作是在内存中进行，速度较Merge Sort Join时要快很多

The classic hash join algorithm for an inner join of two relations proceeds as follows:

First prepare a hash table of the smaller relation. The hash table entries consist of the join attribute and its row. Because the hash table is accessed by applying a hash function to the join attribute, it will be much quicker to find a given join attribute's rows by using this table than by scanning the original relation.
Once the hash table is built, scan the larger relation and find the relevant rows from the smaller relation by looking in the hash table.


（二）索引是什么？有什么作用以及优缺点？

索引是对数据库表中一或多个列的值进行排序的结构

你也可以这样理解：索引就是加快检索表中数据的方法。在数据库中，索引也允许数据库程序迅速地找到表中的数据，而不必扫描整个数据库。

MySQL数据库几个基本的索引类型：普通索引、唯一索引、主键索引、全文索引



（三）什么是事务？

事务（Transaction）是并发控制的基本单位。所谓的事务，它是一个操作序列，这些操作要么都执行，要么都不执行，它是一个不可分割的工作单位。事务是数据库维护数据一致性的单位，在每个事务结束时，都能保持数据一致性。


（四）数据库的乐观锁和悲观锁是什么？

数据库管理系统（DBMS）中的并发控制的任务是确保在多个事务同时存取数据库中同一数据时不破坏事务的隔离性和统一性以及数据库的统一性。

乐观并发控制(乐观锁)和悲观并发控制（悲观锁）是并发控制主要采用的技术手段。

悲观锁：假定会发生并发冲突，屏蔽一切可能违反数据完整性的操作
乐观锁：假设不会发生并发冲突，只在提交操作时检查是否违反数据完整性。






索引的实现通常使用B树及其变种B+树。

在数据之外，数据库系统还维护着满足特定查找算法的数据结构，这些数据结构以某种方式引用（指向）数据，这样就可以在这些数据结构上实现高级查找算法。

为表设置索引要付出代价的：一是增加了数据库的存储空间，二是在插入和修改数据时要花费较多的时间(因为索引也要随之变动)。

一般来说，应该在这些列上创建索引：在经常需要搜索的列上，可以加快搜索的速度；在作为主键的列上，强制该列的唯一性和组织表中数据的排列结构；在经常用在连接的列上，这些列主要是一些外键，可以加快连接的速度；在经常需要根据范围进行搜索的列上创建索引，因为索引已经排序，其指定的范围是连续的；在经常需要排序的列上创建索引，因为索引已经排序，这样查询可以利用索引的排序，加快排序查询时间；在经常使用在WHERE子句中的列上面创建索引，加快条件的判断速度。
