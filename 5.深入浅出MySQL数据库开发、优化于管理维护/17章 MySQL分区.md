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