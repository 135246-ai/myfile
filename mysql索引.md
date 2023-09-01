#### 一、mysql索引

##### 1、联合索引最左前缀法则失效

- 跳过莫一列，索引部分失效，影响的是索引的长度
- 不符合最左前缀，那么会走全表扫描
- 范围查询（<,>）会使后边的列索引失效，主要针对联合索引，<=则不会（这种情况可能会触及走索引如果走全表扫描慢则会优先选择全表扫描（mysql的优化机制-> 数据的分布情况））
- 类型不匹配 int的用字符串表述会使该列失效
- 对列进行函数运算会使该列索引失效 substring()
- like关键字对头部进行模糊查询索引失效
- or分割的条件，一侧有索引一侧没有索引，则不会走索引；解决办法，or两边的条件都建立索引

##### 2、使用原则

- 多个索引指定用那个索引  ----**sql提示**,,在**from**表名之后跟上

  - use index（）建议，具体是否使用mysql内部会进行评估

  - ignore index（）

  - force index（）强制       

- 覆盖索引 & 回表查询

  - extra
    - using index condition; null；查找使用了索引，但需要回表
    - using where;using index；查找使用了索引，所需要的数据在索引列能找到不需要回表

- 前缀索引 （长字符串（varchar、text）考虑使用，会进行回表对查询列判断，把符合条件的返回）

  - create index idx_xxx on table(column(n))   column列 n指前缀的长度

  - count(distinct substring(column,1,9))/count   算前缀长度的公式    区分度

  - Sub_part  为截取的长度

- 单列索引 & 联合索引

  - 建议使用联合索引，防止没有在索引列的字段进行回表查询

    

#### 二、sql优化

##### 1、插入数据

- 批量插入
  - insert into table values(),(),(),... 
  - 一千条以上不建议
- 手动提交事务
  - tart transaction
- 主键顺序插入
- 大批量插入数据，比如百万条
  - 使用mysql数据库提供的load指令
    - 客户端连接服务端时加上参数**--local-infile**       mysql --local-infile  -u root -p
    - 设置全局参数local-infile=1，开启从本地加载文件导入数据的开关   set  global  local-infile=1
    - 执行load指令  load data local infile  ‘数据目录’  into table 'tb_user'  fields  terminated by ','  lines  terminated by '\n'

##### 2、主键优化

- 表数据是根据主键顺序组织存放的，即索引组织表（IOT）
- 逻辑存储结构  tablespace--seqment段--extent区(1M)---page页(16k)--row
- 页分裂  申请内存空间，重新设置链表指针
- 页合并  删除了超过了一个页的50%默认，这个由参数merge_threshold,innodb会开始寻找最靠近页前后可以看看是否可以将两个页合并节省空间
- 尽量减少主键的长度

##### 3、order by优化

- 两种排序方式
  - Using filesort
    - 通过表的索引或全表扫描，读取满足条件的数据行，然后在排序缓冲区sort buffer中完成
  - Using index
    - 通过有序索引顺序扫描直接返回有序数据
- 创建索引默认创建的排序字段都是升序的，可通过explain看
- 严格符合最左前缀法则，因为是有序的，不像and条件，尽量都是用覆盖索引，避免回表查询

##### 4、group by优化

- 建立索引进行优化，避免使用临时表（use temporary）
- 符合最左前缀法则

##### 5、limit优化

- 覆盖索引+子查询
- select  * from tb_user t, (select id from tb_user order by id limit 200000,10) a where t.id=a.id;

##### 6、count优化

- 使用count(*)、count(1)、避免使用count(字段)，会跳过null值。

#### 三、 sql执行计划

- explain ||  desc

  

| id                                                           | select_type                                                  | type                                                         | possible_key   | key            | key_len | rows | filtered                                 | extra |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | -------------- | -------------- | ------- | ---- | ---------------------------------------- | ----- |
| select查询的序列号，表示查询中中执行select子句或者操作表的顺序(id相同，执行顺序从上到下，id不同，值越大，越先执行) | 表示select查询的类型，常见的取值有SIMPLE(简单表，不适用表连接或者子查询)；PRIMARY（主查询，即外层的查询）、UNION（UNION中的第二个或者后面的查询语句）；SUBQUERY（SELECT/WHERE之后包含子查询）等 | 表示连接类型，性能由好到坏的来连接类型为NULL，system、const、eq_ref、ref、range、index、all | 可能用到的索引 | 实际用到的索引 |         |      | 返回结果的行数占所需读取的行数，越大越好 |       |
|                                                              |                                                              |                                                              |                |                |         |      |                                          |       |

