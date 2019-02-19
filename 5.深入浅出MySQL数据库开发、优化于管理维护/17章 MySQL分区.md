# 17章 MySQL分区

## 17.2分区类型

### 17.2.3 columns分区

* 解决的问题：解决了range和list分区只支持整数分区。

* 分类
  * range columns
  * list columns

#### 具体的特点

##### 1.支持多列分区

```mysql
mysql> create table rc3(
    -> a int,
    -> b int)
    -> partition by range columns(a,b) (
    -> partition p01 values less than(0,10),
    -> partition p02 values less than(10,10),
    -> partition p03 values less than(10,20),
    -> partition p04 values less than(20,35)
    -> );
```

* Range columns的分区键的比较是基于元组比较的
* 可以在information_schema库中用如下语句查询数据插入分区的情况：

```sql
select partition_name part,partition_expression expr,partition_description descr,table_rows from partitions where table_name = 'rc3';
```

### 17.2.4Hash分区

* 主要用来分散热点读，确保数据在预先确定个数的分区中尽量平均分配
* 支持两种HASH分区
  * 常规hash分区
  * 线性hash分区
* 常规hash，默认采用mod方式

```mysql
mysql> create table rc4(
    -> id int not null,
    -> ename varchar(30),
    -> hired date not null default '1970-01-01',
    -> separated date not null default '9999-12-31',
    -> job varchar(30) not null,
    -> store_id int not null)
    -> partition by hash(store_id) partitions 4;
```

* 查看执行计划，可以看到具体落入某个分区：

```mysql
explain select * from rc4 where store_id = 113;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | rc4   | p1         | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
```

* 但是如果要增加分区，就很麻烦（如果使用的是取模的分区算法，取模数要变化），这会带来分区管理的麻烦，所以就产生了线性分区(在hash 前面添加了linear)

```mysql
mysql> create table rc5( id int not null, ename varchar(30), hired date not null default '1970-01-01', separated date not null default '9999-12-31', job varchar(30) not null, store_id int not null) partition by linear hash(store_id) partitions 4;
Query OK, 0 rows affected (0.03 sec)
```

* 线性 hash分区的优点是分区维护方便，但是数据没有常规hash分布均匀

### 17.2.5 key分区

*  类似于HASH分区，HASH用户可以自己定义表达式，key分区不可以
* HASH只支持整数分区，key支持除过blob、text外的其他类型作为列的分区键

```sql
mysql> create table rc6(
    -> id int not null,
    -> ename varchar(30),
    -> hired date not null default '1970-01-01',
    -> separated date not null default '9999-12-31',
    -> job varchar(30),
    -> store_id int not null)
    -> partition by key(job) partitions 4;
```

* 如果不指定分区键key，那么会默认使用主键分区，没有主键的时候，会选择非空唯一键（unique）作为分区键（非空），以上都没有的话，就不能指定分区
* key分区也支持在前面使用linear，分区编号通过2的幂运算，二不是取模



### 17.2.6 子分区

* 子分区就是在分区表中再次分区，也叫符合分区
* 支持range或list分区上的hash或者key分区

```sql
mysql> create table rc7(
    -> id int,purchased date)
    -> partition by range(year(purchased))
    -> subpartition by hash(to_days(purchased))
    -> subpartitions 2
    -> ( partition p0 values less than (1990),
    -> partition p1 values less than (2000),
    -> partition p2 values less than MAXVALUE);
```

* 查询的结果

```sql
mysql> select PARTITION_NAME part_name,SUBPARTITION_NAME sub_part ,TABLE_ROWS rows from partitions where table_name = 'rc7';
+-----------+----------+------+
| part_name | sub_part | rows |
+-----------+----------+------+
| p0        | p0sp0    |    0 |
| p0        | p0sp1    |    0 |
| p1        | p1sp0    |    1 |
| p1        | p1sp1    |    3 |
| p2        | p2sp0    |    0 |
| p2        | p2sp1    |    0 |
+-----------+----------+------+
```
### 17.2.7 NUL值的处理
* 在range分区中insert的NULL会当做最小值处理
* list分区insert NULL值时，如果分区中没有包含NULL的分区，报错，如果有NULL值的分区，则写入那个分区

## 17.3 分区管理
### 17.3.1 range和list分区管理
* 在list和range管理相似：
    * 删除分区：alter table table_name drop partition partition_name;删除分区的同时，数据也被删除。
    * 增加分区：alter table table_name add partition (partition partition_name values less than(2000));range分区只能从最大端添加分区，否则报错；增加list分区，增加的分区必须是唯一的，不能喝原有的分区重叠。
    * 合并、拆分分区：mysql提供了不丢失数据的情况下合并和拆分分区的方法。例如将一个range分区的某个分区拆分，将p3分区（2000~2015）拆分成p2分区（2000~2005）和p3（剩下的）。
        ```sql
        alter table table_name reorganize partition p3 into(
        partition p2 values less than (2005),
        partition p3 values less than (2015)
        );
        ```
        合并多个分区为一个分区：
        ```sql
        alter table table_name reorganize partition p1,p2,p3 into (
            partition p1 values less than(2015)
        );
        ```
        重新定义区间的时候，只能定义相邻区间，不能跨越区间，并且新定义的区间要和覆盖原区间，新的区间不能改变分区类型，如不能把range改成hash分区。
### 17.3.2hash和key分区管理
* 不能通过像range和list分区那样删除hash和key分区。
    * coalesce只能减少分区，原分区：
    ```sql
    partition by hash (id) partitions 4;
    ```
    通过如下语句调整分区从4个到2个：

    ```sql
    alter table table_name coalesce partition 2;
    ```
    * 增加分区，曾加8个分区，而不是增加到8个分区：
    ```sql
    alter atble table_name partition partitions 8;
    ```