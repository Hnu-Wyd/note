## Mysql
- [Mysql](#mysql)
  - [B-Tree](#b-tree)
  - [B+树](#b树)
  - [MyISAM存储引擎](#myisam存储引擎)
  - [InnoDB存储引擎](#innodb存储引擎)
  - [Explain工具](#explain工具)
  - [Explain中的列](#explain中的列)
    - [一、id列](#一id列)
    - [二、select_type列](#二select_type列)
    - [三、table列](#三table列)
    - [四、type列](#四type列)
    - [五、possible_keys列](#五possible_keys列)
    - [六、key列](#六key列)
    - [七、key_len列](#七key_len列)
    - [八、ref列](#八ref列)
    - [九、rows列](#九rows列)
    - [十、Extra列](#十extra列)
  - [索引失效](#索引失效)
### B-Tree
B树的特性：
- 叶子节点具有相同的深度，叶子节点的指针为空。
- 不存在重复元素。
- 节点中的元素从左到有依次递增。
![picture 4](img/MySql/Mysql_B-Tree.png)  

### B+树
B+树的特性
- 非叶子节点不存data，只存储索引。
- 叶子节点存储所有索引和data，这样可以存放更多的索引。一般Mysql一次磁盘IO读取16kb的数据，假如是InnoDB存储引擎，我们在数据库的主键一般是整型，占用8个字节，索引指针在数据库中占用6个字节，因此一个整体索引占用14B，在第一层就会有1170个数据，如果默认一个条数据为1B的话，一个节点可以存放16个数据，因此3层B+树可以存放21902400条数据。
- 叶子节点通过双向链表连接，提高区间的访问能力。
![picture 5](img/MySql/Mysql_B+Tree.png)  

### MyISAM存储引擎
在MyISAM存储引擎中一张表被存储为三个文件，一个表结构文件，一个数据文件，一个索引文件。它的数据和索引文件是分离的。
![picture 6](img/MySql/Mysql_MyISAM.png)  
在它以B+树创建的索引是非聚集的，B+树的叶子节点存储的是数据的内存地址。
![picture 7](img/MySql/Mysql_MyISAM_index.png)  

### InnoDB存储引擎
InnoDB存储引擎它的数据和索引是存放在一个文件中的。表文件本身就是一个以B+树组织的索引文件。在它以B+树创建的索引是聚集的，索引的叶子节点包含完整的数据记录。
![picture 9](img/MySql/Mysql_InnoDB_index.png)  

为什么InnoDB表必须有主键，一般主键推荐使用整型自增？
  1. 因为是整型自增的话，插入的数据元素就可以直接插入到索引的后续位置，主键的顺序按照数据记录的插入顺序排列，自动有序。当一页写满，就会自动开辟一个新的页。
  2. 如果是非整型自增，此时MySQL不得不为了将新记录插到合适位置而移动数据，甚至目标页面可能已经被回写到磁盘上而从缓存中清掉，此时又要从磁盘上读回来，这增加了很多开销，同时频繁的移动、分页操作造成了大量的碎片。
  3. 如果设置了主键，那么InnoDB会选择主键作为聚集索引、如果没有显式定义主键，则InnoDB会选择第一个不包含有NULL值的唯一索引作为主键索引、如果也没有这样的唯一索引，则InnoDB会选择内置6字节长的ROWID作为隐含的聚集索引(ROWID随着行记录的写入而主键递增)。

为什么非主键索引（辅助索引）结构的叶子节点存储的是主键值？
   1. 减少了出现行移动或者数据页分裂时二级索引的维护工作（当数据需要更新的时候，二级索引不需要修改，只需要修改聚簇索引，一个表只能有一个聚簇索引，其他的都是二级索引，这样只需要修改聚簇索引就可以了，不需要重新构建二级索引）。
   2. 节省了存储空间。

### Explain工具
Explain可以模拟优化器来执行sql，分析查询语句或者结构的性能瓶颈，在执行查询时mysql会设置一个标记，执行器会返回sql的执行计划，不会真正的执行查询，但是如果from中包含子查询的话，子查询会执行，并把结果放在临时表中。Explain有以下两个变种：
- Explain extended：会在执行计划的基础上，提供一些查询优化的信息，紧随其后增加一个“show warnings”命令可以查询到优化后的sql信息。
- Explain partitions：如果查询的是分区表，此命令会显示将要访问的分区。

### Explain中的列
有如下SQL:
```
1 示例表
2 DROP TABLE IF EXISTS `actor`;
3 CREATE TABLE `actor` (
4 `id` int(11) NOT NULL,
5 `name` varchar(45) DEFAULT NULL,
6 `update_time` datetime DEFAULT NULL,
7 PRIMARY KEY (`id`)
8 ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
9 1
0 INSERT INTO `actor` (`id`, `name`, `update_time`) VALUES (1,'a','2017‐12‐2
2 15:27:18'), (2,'b','2017‐12‐22 15:27:18'), (3,'c','2017‐12‐22 15:27:18');
11
12 DROP TABLE IF EXISTS `film`;
13 CREATE TABLE `film` (
14 `id` int(11) NOT NULL AUTO_INCREMENT,
15 `name` varchar(10) DEFAULT NULL,
16 PRIMARY KEY (`id`),
17 KEY `idx_name` (`name`)
18 ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
19
20 INSERT INTO `film` (`id`, `name`) VALUES (3,'film0'),(1,'film1'),(2,'film
2');
21
22 DROP TABLE IF EXISTS `film_actor`;
23 CREATE TABLE `film_actor` (
24 `id` int(11) NOT NULL,
25 `film_id` int(11) NOT NULL,
26 `actor_id` int(11) NOT NULL,
27 `remark` varchar(255) DEFAULT NULL,
28 PRIMARY KEY (`id`),
29 KEY `idx_film_actor_id` (`film_id`,`actor_id`)30 ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
31
32 INSERT INTO `film_actor` (`id`, `film_id`, `actor_id`) VALUES (1,1,1),
(2,1,2),(3,2,1);
```
#### 一、id列
id的编号是select的序列号，有几个select就有几个id，id越大优先级越高越先执行，id相同依次从上到下执行，id为null则最后执行。

#### 二、select_type列
1. simple：简单查询，查询不包含子查询和union。如(select * from film where id = 2)。
2. primary：复杂查询的最外层查询。
3. subquery：包含在select中的子查询，不在from子句中，在外层from的前面。
4. derived：包含在from子查询中的select。mysql会把子查询的查询结果放在一个临时表里面。临时表也称为派生表（derived的英文含义）。
（关闭mysql5.7的对衍生表合并的新特性，```set session optimizer_switch='derived_merge=off'```执行以下查询语句```explain select (select 1 from actor where id = 1) from (select * from film where id = 1) der;```）
![picture 10](img/MySql/Explain_select_type.png)  

5. union:在union中第二个select。如sql:````explain select 1 union all select 1;````
![picture 11](img/MySql/Explain_select_type_union.png)  


#### 三、table列
table列表示执行select访问的哪张表。当from查询有子查询的时候table列是````<deriver N> ````N表示当前查询依赖```id=N```的查询。当有union时，```<union 1,2>```,1和2表参与union的select行的id。

#### 四、type列
表示关联类型或者访问类型，即mysql决定要如何进行查找。从最优到最差依次是```null > system > const > eq_ref > ref > rang > index > all``` 一般的sql查询要优化到rang基别，最好是到ref。
- null：在查询的时候不用访问表或者索引。
```select 1 from film where 1```
- consta, system：单表中最多有一个匹配行，查询起来非常快，一般根据主键和唯一索引查询，system是const的特列，表中只有一行数据。
- eq_ref：使用主键或唯一索引来进行扫描，对于每个索引键表中只有一条记录与之匹配。
- ref：使用普通索引或者普通索引的前缀进行扫描。
- rang：扫描部分索引，索引的范围查询，常见于```between，<，>```等查询。
- index：查询全部索引树。
- all：全表扫描。

#### 五、possible_keys列
表示查询可能使用到的所有，假如为此列有值，而key列为空时，表示表的数据不多，mysql任务使用所有对查询的帮助不大，因此选择了全表扫描，如果此列为空，需要优化sql，查看where条件是否可以创造个索引。

#### 六、key列
表示实际查询中使用的那个索引，如果没有索引则为空，如果想强制使用或者忽略```possible_keys```列中的索引，可以在查询的时候加上```force index、ignore index```。

#### 七、key_len列
表示使用索引字段的长度，通过这个值可以算出具体使用了索引中的哪些字段。
key_len计算规则如下：
- 字符串
    - char(n)：n字节长度
    - varchar(n)：2字节存储字符串长度，如果是utf-8，则长度 3n
- 数值类型
    - tinyint：1字节
    - smallint：2字节
    - int：4字节
    - bigint：8字节
- 时间类型
    - date：3字节
    - timestamp：4字节
    - datetime：8字节
  
如果字段允许为 NULL，需要1字节记录是否为 NULL，索引最大长度是768字节，当字符串过长时，mysql会做一个类似左前缀索引的处理，将前半部分的字符提取出来做索引。

#### 八、ref列
表示哪个字段或常数与key一起被使用。

#### 九、rows列
表示要遍历多少数据才能找到，估算的找到所需记录需要读取的行数，是个估算值，并不准确。

#### 十、Extra列
- using index：使用覆盖索引，不用查询表数据，只用索引就可以完成查询。
- using where：查询的列未被索引覆盖。
- using index condition：查询的列不完全被索引覆盖，where条件是一个前导列的范围。
- using temporary：需要创建一张临时表来处理查询，一般需要优化。
- using filesort：将用外部排序而不是索引排序，数据较小在内存排序，否则在磁盘完成排序，一般需要优化。
- Select tables optimized away：使用某些聚合函数（比如 max、min）来访问存在索引的某个字段时。

### 索引失效
- 最左前缀法则：带头大哥不能死，中间兄弟不能掉。如果不遵循词法则会导致索引失效。
- 在索引上做计算、函数、自动或手动类型转换会导致索引失效。
- 使用 ```!=、=、<、>```无法使用索引。
- is null、is not null也无法使用索引。
- like查询使用通配符（```$,%```）开头会使索引失效。
- 字符串不加单引号会使索引失效。
- 尽量少使用```in、or```，用它查询时不一定会走索引，mysql内部会根据表大小、索引比例等多个因素来评估是否使用索引。

尽量使用覆盖索引，减少```select *``` 的使用。















