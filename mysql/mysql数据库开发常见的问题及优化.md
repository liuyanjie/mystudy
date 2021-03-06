# mysql 数据库开发常见问题及优化

## 一、库表设计

### 1.1 引擎选择

`mysql`常用的存储引擎包括`MYISAM`、`Innodb`和`Memory`，其中各自的特点如下：

- `MYISAM`：全表锁，拥有较高的执行速度，一个写请求请阻塞另外相同表格的所有读写请求，并发性能差，占用空间相对较小，`mysql 5.5`及以下仅`MYISAM`支持全文索引，不支持事务。

- `Innodb`：行级锁（`SQL`都走索引查询），并发能力相对强，占用空间是`MYISAM`的2.5倍，不支持全文索引（`mysql5.6`开始支持），支持事务。

- `Memory`：全表锁，存储在内存当中，速度快，但会占用和数据量成正比的内存空间且数据在`mysql`重启时会丢失。

基于以上特性，建议绝大部份都设置为`innodb`引擎，特殊的业务再考虑选用`MYISAM`或`Memory`，如全文索引支持或极高的执行效率等。

### 1.2 分表方法

在数据库表使用过程中，为了减小数据库服务器的负担、缩短查询时间，常常会考虑做分表设计。分表分两种，一种是纵向分表（将本来可以在同一个表的内容，人为划分存储在为多个不同结构的表）和横向分表（把大的表结构，横向切割为同样结构的不同表）。

纵向分表常见的方式有根据活跃度分表、根据重要性分表等。其主要解决问题如下：

- 表与表之间资源争用问题；
- 锁争用机率小；
- 实现核心与非核心的分级存储；
- 解决了数据库同步压力问题。

横向分表是指根据某些特定的规则来划分大数据量表，如根据时间分表。其主要解决问题如下：

- 单表过大造成的性能问题；
- 单表过大造成的单服务器空间问题。

### 1.3 表的设计

- 选择合适的数值类型，勿设置太大。
- 尽可能的使用`varchar/nvarchar`代替`char/nchar`。
  1. 变长字段存储空间小，可以节省存储空间
  2. 对于查询来说，在一个相对较小的字段内搜索效率显然要高些。
- 尽量使用数字型字段
  1. 若只含数值信息的字段尽量不要设计为字符型，这会降低查询和连接的性能，并会增加存储开销。
  2. 字符转化为数字，如用int而不是char(15)存储ip
  3. 这是因为引擎在处理查询和连接时会 逐个比较字符串中每一个字符，而对于数字型而言只需要比较一次就够了。
- 优先使用enum或set
- 避免使用NULL字段。NULL字段很难查询优化；NULL字段的索引需要额外空间；NULL字段的复合索引无效；
- 少用text。varchar的性能会比text高很多；
- 同一意义的字段定义必须相同，比如不同表中都有uid字段，那么它的类型、字段长度要设计成一样

### 1.4 索引问题

索引是对数据库表中一个或多个列的值进行排序的结构，建立索引有助于更快地获取信息。 `mysql`有四种不同的索引类型：

1. 主键索此 (`PRIMARY`)
2. 唯一索引 (`UNIQUE`)
3. 普通索引 (`INDEX`)
4. 全文索引（`FULLTEXT`, `MYISAM`及`mysql 5.6`以上的`Innodb`）

建立索引的目的是加快对表中记录的查找或排序，索引并不是越多越好，因为创建索引是要付出代价的：一是增加了数据库的存储空间，二是在插入和修改数据时要花费较多的时间维护索引。一个表的索引数最好不要超过6个，若太多则应考虑一些不常使用到的列上建的索引是否有必要。

在设计表或索引时，常出现以下几个问题：

- 少建索引或不建索引。
- 索引滥用。滥用索引将导致写请求变慢，拖慢整体数据库的响应速度。
- 从不考虑联合索引。实际上联合索引的效率往往要比单列索引的效率更高。
- 非最优列选择。低选择性的字段不适合建单列索引，如只有0、1标识的状态字段。

## 二、优化目标
1. 减少`IO`次数
  `IO`永远是数据库最容易瓶颈的地方，这是由数据库的职责所决定的，大部分数据库操作中超过90%的时间都是`IO`操作所占用的，减少`IO`次数是`SQL`优化中需要第一优先考虑，当然，也是收效最明显的优化手段。
2. 降低`CPU`计算
  除了`IO`瓶颈之外，`SQL`优化中需要考虑的就是`CPU`运算量的优化了。`ORDER BY, GROUP BY, DISTINCT` … 都是消耗`CPU`的大户（这些操作基本上都是`CPU`处理内存中的数据比较运算）。当我们的`IO`优化做到一定阶段之后，降低 `CPU` 计算也就成为了我们`SQL`优化的重要目标

## 三、慢`SQL`问题

### 导致慢`SQL`的原因

在遇到慢`SQL`情况时，不能简单的把原因归结为`SQL`编写问题(虽然这是最常见的因素)，实际上导致慢`SQL`有很多因素，甚至包括硬件和 `mysql`本身的`bug`。根据出现的概率从大到小，罗列如下：

- `SQL`编写问题
- 锁
- 业务实例相互干绕对`IO/CPU`资源争用
- 服务器硬件
- `MYSQL BUG`

### 由`SQL`编写导致的慢`SQL`优化

针对`SQL`编写导致的慢，优化起来还是相对比较方便的。正如上一节提到的正确的使用索引能加快查询速度，那么我们在编写`SQL`语句的时候就需要注意与索引相关的规则：

- 字段类型转换导致不用索引，如字符串类型的不用引号，数字类型的用引号等，这有可能会用不到索引导致全表扫描；
- 不要在索引列参与计算、不要索引列使用了函数；
- 字符串比较长的可以考虑索引一部份减少索引文件大小，提高写入效率；
- 根据联合索引的第二个及以后的字段单独查询用不到索引；
- `IS NULL, <>, !=, !>, !<, NOT, NOT EXISTS, NOT IN, NOT LIKE, LIKE '%500'`用不到索引
- 不要使用`SELECT *`；
- 排序请尽量使用升序、尽量少排序、排序的列上建立索引;
- `OR`的查询尽量用`UNION`代替（`Innodb`）；
- 复合索引高选择性的字段排在前面；
- `ORDER BY, GROUP BY`字段包括在索引当中减少排序，效率会更高。

除了上述索引使用规则外，`SQL`编写时还需要特别注意一下几点：

- 尽量规避大事务的`SQL`，大事务的`SQL`会影响数据库的并发性能及主从同步；
- 分页语句`LIMIT`的问题；
- 删除表所有记录请用`TRUNCATE`，不要用`DELETE`；
- 不让`mysql`干多余的事情，如计算；
- 输写`SQL`带字段，以防止后面表变更带来的问题，性能也是比较优的；
- 在`Innodb`上用`SELECT COUNT(*)`，因为`Innodb`会存储统计信息；
- 慎用`ORDER BY RAND()`。

## 四、分析诊断工具

在日常开发工作中，我们可以做一些工作达到预防慢`SQL`问题，比如在上线前预先用诊断工具对`SQL`进行分析。常用的工具有：

```
    mysqldumpslow
    mysql profile
    mysql explain
```

## 五、优化的详细点

1. 查询有索引的字段优先，都有索引的看查询出来的数据量，少的优先
  `WHERE`子句后面的条件顺序对大数据量表的查询会产生直接的影响，如

  ```sql
   SELECT * FROM tbl_posts WHERE t_gid = 143 AND t_is_publish = 12;
   SELECT * FROM tbl_posts WHERE t_is_publish = 12 AND t_gid = 143;
  ```

  如以上两个`SQL`语句，条件为`t_gid = 143`数据少一些，则此条件放在前边，第一天语句效率和资源cpu等占用比第二天低。

2. 选择最有效率的表名顺序
`sql`解析器按照从右到左的顺序处理`FROM`子句中的表名，因此`FROM`子句中写在最后的表将被最先处理。在`FROM`子句中包含多个表的情况下，你必须选择记录条数最少的表作为基础表。 

3. 查询时尽量不要返回不需要的行、列。另外在多表连接查询时，尽量改成连接查询，少用子查询。 

4. `BETWEEN`在某些时候比`IN`速度更快，`BETWEEN`能够更快地根据索引找到范围。用查询优化器可见到差别。 

  ```sql
    SELECT * FROM tbl_user WHERE sex IN ('男','女') 
    SELECT * FROM tbl_user WHERE BETWEEN '男' AND '女' // 是一样的。由于IN会在比较多次，所以有时会慢些。
  ```

5. 用`OR`的查询尽量用`UNION`代替。
   他们的速度只同是否使用索引有关，如果查询需要用到联合索引，用 `UNION ALL`执行的效率更高。
   多个`OR`的字句没有用到索引，改写成`UNION`的形式再试图与索引匹配。
   注意`UNION`和`UNION ALL`的区别。`UNION`比`UNION ALL`多做了一步`DISTINCT`操作。能用`UNION ALL`的情况下尽量不用`UNION`。

6. 使用`IN`时，在`IN`后面值的列表中，将出现最频繁的值放在最前面，出现得最少的放在最后面，这样可以减少判断的次数 

7. 一般在`GROUP BY`和`HAVING`字句之前就能剔除多余的行，所以尽量不要用它们来做剔除行的工作。他们的执行顺序应该如下最优：`SELECT`的`WHERE`字句选择所有合适的行，`GROUP BY`用来分组个统计行，`HAVING`字句用来剔除多余的分组。这样`GROUP BY`和`HAVING`的开销小，查询快。对于大的数据行进行分组和`HAVING`十分消耗资源。如果`GROUP BY`的目的不包括计算，只是分组，那么用`DISTINCT`更快。

8. 一次更新多条记录比分多次更新每次一条快，批处理好

9. 应尽可能的避免更新聚集索引数据列，因为聚集索引数据列的顺序就是表记录的物理存储顺序，一旦该列值改变将导致整个表记录的顺序的调整，会耗费相当大的资源。若应用系统需要频繁更新聚集索引数据列，那么需要考虑是否应将该索引建为聚集索引。

10. 尽量少用视图，它的效率低。对视图操作比直接对表操作慢，可以用存储过程来代替它。特别的是不要用视图嵌套，嵌套视图增加了寻找原始资料的难度。我们看视图的本质：它是存放在服务器上的被优化好了的已经产生了查询规划的`SQL`。对单个表检索数据时，不要使用指向多个表的视图，
直接从表检索或者仅仅包含这个表的视图上读，否则增加了不必要的开销，查询受到干扰。

11. 在必要时对全局或者局部临时表创建索引，有时能够提高速度，但不是一定会这样，因为索引也耗费大量的资源。他的创建同是实际表一样。

12. 尽量使用表变量来代替临时表
临时表存储于`tempdb`库中，操作临时表时，会引起跨库操作。尽量用结果集和表变量来代替它。如果表变量包含大量数据，请注意索引非常有限（只有主键索引）。避免频繁创建和删除临时表，以减少系统表资源的消耗。

13. 临时表并不是不可使用，适当地使用它们可以使某些例程更有效，例如，当需要重复引用大型表或常用表中的某个数据集时。但是，对于一次性事件，最好使用导出表。

14. 在新建临时表时，如果一次性插入数据量很大，那么可以使用`select into`代替`create table`，避免造成大量`log`，以提高速度；如果数据量不大，为了缓和系统表的资源，应先`create table`，然后`insert`。

15. 如果使用到了临时表，在存储过程的最后务必将所有的临时表显式删除，先`truncate table`，然后`drop table`，这样可以避免系统表的较长时间锁定。

16. 尽量将数据的处理工作放在服务器上，减少网络的开销，如使用存储过程。存储过程是编译好、优化过，并且被组织到一个执行规划里、且存储在数据库中的`SQL`语句，是控制流语言的集合，速度当然快。 

17. 不要在一段`SQL`或者存储过程中多次使用相同的函数或相同的查询语句，这样比较浪费资源，建议将结果放在变量里再调用，这样更快。 

18. 按照一定的次序来访问你的表。如果你先锁住表`A`，再锁住表`B`，那么在所有的存储过程中都要按照这个顺序来锁定它们。如果在某个存储过程中先锁定表`B`，再锁定表`A`，这可能就会导致一个死锁。如果锁定顺序没有被预先详细的设计好，死锁很难被发现。

19. 尽量避免使用游标，因为游标的效率较差，如果游标操作的数据超过1万行，那么就应该考虑改写。

20. 使用基于游标的方法或临时表方法之前，应先寻找基于集的解决方案来解决问题，基于集的方法通常更有效。

21. 与临时表一样，游标并不是不可使用。对小型数据集使用`FAST_FORWARD`游标通常要优于其他逐行处理方法，尤其是在必须引用几个表才能获得所需的数据时。在结果集中包括“合计”的例程通常要比使用游标执行的速度快。如果开发时 间允许，基于游标的方法和基于集的方法都可以尝试一下，看哪一种方法的效果更好。

22. 在所有的存储过程和触发器的开始处设置`SET NOCOUNT ON`，在结束时设置`SET NOCOUNT OFF`。无需在执行存储过程和触发器的每个语句后向客户端发送`DONE_IN_PROC`消息。

23. 尽量避免向客户端返回大数据量，若数据量过大，应该考虑相应需求是否合理。

24. 尽量避免大事务操作，提高系统并发能力。