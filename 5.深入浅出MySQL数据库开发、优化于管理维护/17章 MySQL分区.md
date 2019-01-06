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

