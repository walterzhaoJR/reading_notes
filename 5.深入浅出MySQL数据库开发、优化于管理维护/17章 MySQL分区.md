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

