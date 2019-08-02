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
