## 一、分区表的分类与限制



### 1、分区表分类

**RANGE分区**：基于属于一个给定连续区间的列值，把多行分配给分区。

**LIST分区**：类似于按RANGE分区，区别在于LIST分区是基于列值匹配一个离散值集合中的某个值来进行选择。

**HASH分区**：基于用户定义的表达式的返回值来进行选择的分区，该表达式使用将要插入到表中的这些行的列值进行计算。这个函数可以包含MySQL 中有效的、产生非负整数值的任何表达式。

**KEY分区**：类似于按HASH分区，区别在于KEY分区只支持计算一列或多列，且MySQL服务器提供其自身的哈希函数。必须有一列或多列包含整数值。

**复合分区**：在MySQL 5.6版本中，只支持RANGE和LIST的子分区，且子分区的类型只能为HASH和KEY。



### 2、分区表限制

1）分区键必须包含在表的所有主键、唯一键中。

2）MYSQL只能在使用分区函数的列本身进行比较时才能过滤分区，而不能根据表达式的值去过滤分区，即使这个表达式就是分区函数也不行。

3）最大分区数： 不使用NDB存储引擎的给定表的最大可能分区数为8192（包括子分区）。如果当分区数很大，但是未达到8192时提示 Got error … from storage engine: Out of resources when opening file,可以通过增加open_files_limit系统变量的值来解决问题，当然同时打开文件的数量也可能由操作系统限制。

4）不支持查询缓存： 分区表不支持查询缓存，对于涉及分区表的查询，它自动禁用。 查询缓存无法启用此类查询。

5）分区的innodb表不支持外键。

6）服务器SQL_mode影响分区表的同步复制。 主机和从机上的不同SQL_mode可能会导致sql语句; 这可能导致分区之间的数据分配给定主从位置不同，甚至可能导致插入主机上成功的分区表在从库上失败。 为了获得最佳效果，您应该始终在主机和从机上使用相同的服务器SQL模式。

7）ALTER TABLE … ORDER BY： 对分区表运行的ALTER TABLE … ORDER BY列语句只会导致每个分区中的行排序。

8）全文索引。 分区表不支持全文索引，即使是使用InnoDB或MyISAM存储引擎的分区表。

9）分区表无法使用外键约束。

10）Spatial columns： 具有空间数据类型（如POINT或GEOMETRY）的列不能在分区表中使用。

11）临时表： 临时表不能分区。

12）subpartition问题： subpartition必须使用HASH或KEY分区。 只有RANGE和LIST分区可能被分区; HASH和KEY分区不能被子分区。

13）分区表不支持mysqlcheck，myisamchk和myisampack。

 

## 二、创建分区表



### **1、range分区**

行数据基于一个给定的连续区间的列值放入分区。

```
CREATE TABLE `test_11` (
 `id` int(11) NOT NULL,
 `t` date NOT NULL,
 PRIMARY KEY (`id`,`t`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
 PARTITION BY RANGE (to_days(t))
(PARTITION p20170801 VALUES LESS THAN (736907) ENGINE = InnoDB,
 PARTITION p20170901 VALUES LESS THAN (736938) ENGINE = InnoDB,
 PARTITION pmax VALUES LESS THAN maxvalue ENGINE = InnoDB);123456789
```

然后插入4条数据：

```
insert into test_11 values (1,"20170722"),(2,"20170822"),(3,"20170823"),(4,"20170824");
```

然后查看information下partitions对分区别信息的统计：

```
select PARTITION_NAME as "分区",TABLE_ROWS as "行数" from information_schema.partitions where table_schema="mysql_test" and table_name="test_11";
+-----------+--------+
| 分区   | 行数  |
+-----------+--------+
| p20170801 |   1 |
| p20170901 |   3 |
+-----------+--------+
2 rows in set (0.00 sec)12345678
```

可以看出分区p20170801插入1行数据，p20170901插入的3行数据。 
可以是用year、to_days、unix_timestamp等函数对相应的时间字段进行转换，然后分区。



### **2、list分区**

和range分区一样，只是list分区面向的是离散的值

```
CREATE TABLE h2 ( 
 c1 INT,
 c2 INT
)
PARTITION BY LIST(c1) (
 PARTITION p0 VALUES IN (1, 4, 7),
 PARTITION p1 VALUES IN (2, 5, 8)
);

#结果
Query OK, 0 rows affected (0.11 sec)123456789
```

与RANGE分区的情况不同，没有“catch-all”，如MAXVALUE; 分区表达式的所有预期值应在PARTITION … VALUES IN（…）子句中涵盖。 包含不匹配的分区列值的INSERT语句失败并显示错误，如此示例所示：

```
INSERT INTO h2 VALUES (3, 5);
#报错提示
ERROR 1525 (HY000): Table has no partition for value 312
```

### **3、hash分区**

根据用户自定义表达式的返回值来进行分区，返回值不能为负数

```
CREATE TABLE t1 (col1 INT, col2 CHAR(5), col3 DATE)
  PARTITION BY HASH( YEAR(col3) )
  PARTITIONS 4;
```

如果你插入col3的数值为’2005-09-15’，那么根据以下计算来选择插入的分区：

```
MOD(YEAR('2005-09-01'),4)
  = MOD(2005,4)
  = 1123
```



### **4、key分区**

根据MySQL数据库提供的散列函数进行分区

```
CREATE TABLE k1 (
  id INT NOT NULL,
  name VARCHAR(20),
  UNIQUE KEY (id)
)
PARTITION BY KEY()
PARTITIONS 2;
```

KEY仅列出零个或多个列名称。 用作分区键的任何列必须包含表的主键的一部分或全部，如果该表具有一个。 如果没有列名称作为分区键，则使用表的主键（如果有）。如果没有主键，但是有一个唯一的键，那么唯一键用于分区键。但是，如果唯一键列未定义为NOT NULL，则上一条语句将失败。

与其他分区类型不同，KEY使用的分区不限于整数或空值。 例如，以下CREATE TABLE语句是有效的：

```
CREATE TABLE tm1 (
  s1 CHAR(32) PRIMARY KEY
)
PARTITION BY KEY(s1)
PARTITIONS 10;
```

注意：对于key分区表，不能执行ALTER TABLE DROP PRIMARY KEY，因为这样做会生成错误 ERROR 1466 (HY000): Field in list of fields for partition function not found in table. 



### **5、Column分区**

COLUMN分区是5.5开始引入的分区功能，只有RANGE COLUMN和LIST COLUMN这两种分区；支持整形、日期、字符串；RANGE和LIST的分区方式非常的相似。

COLUMNS和RANGE和LIST分区的区别
1）针对日期字段的分区就不需要再使用函数进行转换了，例如针对date字段进行分区不需要再使用YEAR()表达式进行转换。
2）COLUMN分区支持多个字段作为分区键但是不支持表达式作为分区键。

column支持的数据类型：
1）所有的整型，float和decimal不支持
2）日期类型：date和datetime，其他不支持
3）字符类型：CHAR, VARCHAR, BINARY和VARBINARY，blob和text不支持 
单列的column range分区

```
 CREATE TABLE `list_c` ( 
  `c1` int(11) DEFAULT NULL, 
  `c2` int(11) DEFAULT NULL 
) ENGINE=InnoDB DEFAULT CHARSET=latin1 
PARTITION BY RANGE  COLUMNS(c1) 
(PARTITION p0 VALUES LESS THAN (5) ENGINE = InnoDB, 
PARTITION p1 VALUES LESS THAN (10) ENGINE = InnoDB);
```

多列的column range分区

```
CREATE TABLE `list_c` (
 `c1` int(11) DEFAULT NULL,
 `c2` int(11) DEFAULT NULL,
 `c3` char(20) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=latin1
PARTITION BY RANGE COLUMNS(c1,c3)
(PARTITION p0 VALUES LESS THAN (5,'aaa') ENGINE = InnoDB,
 PARTITION p1 VALUES LESS THAN (10,'bbb') ENGINE = InnoDB) ;
```


单列的column list分区

```
CREATE TABLE `list_c` (
 `c1` int(11) DEFAULT NULL,
 `c2` int(11) DEFAULT NULL,
 `c3` char(20) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=latin1
PARTITION BY LIST COLUMNS(c3)
(PARTITION p0 VALUES IN ('aaa') ENGINE = InnoDB,
 PARTITION p1 VALUES IN ('bbb') ENGINE = InnoDB) ;
```

 



### **6、子分区（组合分区）**

在分区的基础上再进一步分区，有时成为复合分区;

MySQL数据库允许在range和list的分区上进行HASH和KEY的子分区。例如：

```
CREATE TABLE ts (id INT, purchased DATE)
  PARTITION BY RANGE( YEAR(purchased) )
  SUBPARTITION BY HASH( TO_DAYS(purchased) )
  SUBPARTITIONS 2 (
    PARTITION p0 VALUES LESS THAN (1990),
    PARTITION p1 VALUES LESS THAN (2000),
    PARTITION p2 VALUES LESS THAN MAXVALUE
  );
```

```
[root@mycat-3 ~]# ll /data/mysql_data_3306/mysql_test/ts*
-rw-r----- 1 mysql mysql 8596 Aug 8 13:54 /data/mysql_data_3306/mysql_test/ts.frm
-rw-r----- 1 mysql mysql 98304 Aug 8 13:54 /data/mysql_data_3306/mysql_test/ts#P#p0#SP#p0sp0.ibd
-rw-r----- 1 mysql mysql 98304 Aug 8 13:54 /data/mysql_data_3306/mysql_test/ts#P#p0#SP#p0sp1.ibd
-rw-r----- 1 mysql mysql 98304 Aug 8 13:54 /data/mysql_data_3306/mysql_test/ts#P#p1#SP#p1sp0.ibd
-rw-r----- 1 mysql mysql 98304 Aug 8 13:54 /data/mysql_data_3306/mysql_test/ts#P#p1#SP#p1sp1.ibd
-rw-r----- 1 mysql mysql 98304 Aug 8 13:54 /data/mysql_data_3306/mysql_test/ts#P#p2#SP#p2sp0.ibd
-rw-r----- 1 mysql mysql 98304 Aug 8 13:54 /data/mysql_data_3306/mysql_test/ts#P#p2#SP#p2sp1.ibd
1234567891011121314151617
```

ts表根据purchased进行range分区，然后又进行了一次hash分区，最后形成了3*2个分区，可以从物理文件证实此分区方式。可以通过subpartition语法来显示指定子分区名称。

注意：每个子分区的数量必须相同；如果一个分区表的任何子分区已经使用subpartition，那么必须表明所有的子分区名称；每个subpartition子句必须包括子分区的一个名字；子分区的名字必须是一致的

另外，对于MyISAM表可以使用index directory和data direactory来指定各个分区的数据和索引目录，但是对于innodb表来说，因为该存储引擎使用表空间自动的进行数据和索引的管理，因此会忽略指定index和data的语法。

 



## 三、普通表转换为分区表

1、用alter table table_name partition by命令重建分区表

alter table jxfp_data_bak PARTITION BY KEY(SH) PARTITIONS 8;

 

## 四、分区表操作



```
CREATE TABLE t1 (
  id INT,
  year_col INT
)
PARTITION BY RANGE (year_col) (
  PARTITION p0 VALUES LESS THAN (1991),
  PARTITION p1 VALUES LESS THAN (1995),
  PARTITION p2 VALUES LESS THAN (1999)
);
```



### 1、ADD PARTITION （新增分区）

```
ALTER TABLE t1 ADD PARTITION (PARTITION p3 VALUES LESS THAN (2002));
```



### 2、DROP PARTITION （删除分区）

```
ALTER TABLE t1 DROP PARTITION p0, p1;
```



### 3、TRUNCATE PARTITION（截取分区）

```
ALTER TABLE t1 TRUNCATE PARTITION p0;

ALTER TABLE t1 TRUNCATE PARTITION p1, p3;

```



### 4、COALESCE PARTITION（合并分区）

```
CREATE TABLE t2 (
  name VARCHAR (30),
  started DATE
)
PARTITION BY HASH( YEAR(started) )
PARTITIONS 6;

ALTER TABLE t2 COALESCE PARTITION 2;
```



### 5、REORGANIZE PARTITION（拆分/重组分区）

1）拆分分区

```
ALTER TABLE table ALGORITHM=INPLACE, REORGANIZE PARTITION;

ALTER TABLE employees ADD PARTITION (
  PARTITION p5 VALUES LESS THAN (2010),
  PARTITION p6 VALUES LESS THAN MAXVALUE
);
```

2）重组分区

```
ALTER TABLE members REORGANIZE PARTITION s0,s1 INTO (
  PARTITION p0 VALUES LESS THAN (1970)
);
ALTER TABLE tbl_name
  REORGANIZE PARTITION partition_list
  INTO (partition_definitions);
ALTER TABLE members REORGANIZE PARTITION p0,p1,p2,p3 INTO (
  PARTITION m0 VALUES LESS THAN (1980),
  PARTITION m1 VALUES LESS THAN (2000)
);
ALTER TABLE tt ADD PARTITION (PARTITION np VALUES IN (4, 8));
ALTER TABLE tt REORGANIZE PARTITION p1,np INTO (
  PARTITION p1 VALUES IN (6, 18),
  PARTITION np VALUES in (4, 8, 12)
);
```



### 6、ANALYZE 、CHECK PARTITION（分析与检查分区）

1）ANALYZE 读取和存储分区中值的分布情况

```
ALTER TABLE t1 ANALYZE PARTITION p1, ANALYZE PARTITION p2;

ALTER TABLE t1 ANALYZE PARTITION p1, p2;
```

2）CHECK 检查分区是否存在错误

```
ALTER TABLE t1 ANALYZE PARTITION p1, CHECK PARTITION p2;
```



### 7、REPAIR分区

修复被破坏的分区

```
ALTER TABLE t1 REPAIR PARTITION p0,p1;
```



### 8、OPTIMIZE

该命令主要是用于回收空闲空间和分区的碎片整理。对分区执行该命令,相当于依次对分区执行 **CHECK PARTITION, ANALYZE PARTITION,REPAIR PARTITION**命令。

譬如:

```
ALTER TABLE t1 OPTIMIZE PARTITION p0, p1;
```



### 9、REBUILD分区

重建分区,它相当于先删除分区中的数据,然后重新插入。这个主要是用于分区的碎片整理。

```
ALTER TABLE t1 REBUILD PARTITION p0, p1;
```



### 10、EXCHANGE PARTITION（分区交换）

分区交换的语法如下:

ALTER TABLE pt EXCHANGE PARTITION p WITH TABLE nt

其中,pt是分区表,p是pt的分区(注:也可以是子分区),nt是目标表。

其实,分区交换的限制还是蛮多的:

1） nt不能为分区表

2）nt不能为临时表

3）nt和pt的结构必须一致

4）nt不存在任何外键约束,即既不能是主键,也不能是外键。

5）nt中的数据不能位于p分区的范围之外。

具体可参考MySQL的官方文档



### 11、迁移分区（DISCARD 、IMPORT ）

```
ALTER TABLE t1 DISCARD PARTITION p2, p3 TABLESPACE;

ALTER TABLE t1 IMPORT PARTITION p2, p3 TABLESPACE;
```

实验环境：（都是mysql5.7）
源库：192.168.2.200   mysql5.7.16  zhangdb下的emp_2分区表的 
目标库：192.168.2.100  mysql5.7.18  test下 （将zhangdb的emp表，导入到目标库的test schema下）

--：在源数据库中创建测试分区表emp_2，然后导入数据

```
 CREATE TABLE emp_2(
id BIGINT unsigned NOT NULL AUTO_INCREMENT,
x VARCHAR(500) NOT NULL,
y VARCHAR(500) NOT NULL,
PRIMARY KEY(id)
)
PARTITION BY RANGE COLUMNS(id) 
(
PARTITION p1 VALUES LESS THAN (1000), 
PARTITION p2 VALUES LESS THAN (2000), 
PARTITION p3 VALUES LESS THAN (3000)   
); 
```

（接着创建存储过程，导入测试数据）

```
DELIMITER //
CREATE PROCEDURE insert_batch()
begin 
DECLARE num INT;
SET num=1;
WHILE num < 3000 DO
IF (num%10000=0) THEN
COMMIT;
END IF;
INSERT INTO emp_2 VALUES(NULL, REPEAT('X', 500), REPEAT('Y', 500));
SET num=num+1;
END WHILE;
COMMIT;
END //

DELIMITER ;
```



```
select TABLE_NAME,PARTITION_NAME from information_schema.partitions where table_schema='zhangdb';
+------------+----------------+
| TABLE_NAME | PARTITION_NAME |
+------------+----------------+
| emp    | NULL      |
| emp_2   | p1       |
| emp_2   | p2       |
| emp_2   | p3       |
+------------+----------------+
4 rows in set (0.00 sec)
```

 

```
select count(*) from emp_2 partition (p1);
+----------+
| count(*) |
+----------+
|   999 |
+----------+
1 row in set (0.00 sec)

mysql> select count(*) from emp_2 partition (p2);
+----------+
| count(*) |
+----------+
|   1000 |
+----------+
1 row in set (0.00 sec)
```

```
 select count(*) from emp_2 partition (p3);
+----------+
| count(*) |
+----------+
|   1000 |
+----------+
1 row in set (0.00 sec)
```

从上面可以看出，emp_2分区表已经创建完成，并且有3个子分区，每个分区都有一点数据。

--：在目标数据库中，创建emp_2表的结构，不要数据（要在源库，使用show create table emp_2\G 的方法 查看创建该表的sql）

```
show create table emp_2\G


CREATE TABLE `emp_2` (
 `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
 `x` varchar(500) NOT NULL,
 `y` varchar(500) NOT NULL,
 PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=3000 DEFAULT CHARSET=utf8mb4
/*!50500 PARTITION BY RANGE COLUMNS(id)
(PARTITION p1 VALUES LESS THAN (1000) ENGINE = InnoDB,
 PARTITION p2 VALUES LESS THAN (2000) ENGINE = InnoDB,
 PARTITION p3 VALUES LESS THAN (3000) ENGINE = InnoDB) */ ;
```

```
[root@localhost test]# ll
-rw-r----- 1 mysql mysql 98304 May 25 15:58 emp_2#P#p0.ibd
-rw-r----- 1 mysql mysql 98304 May 25 15:58 emp_2#P#p1.ibd
-rw-r----- 1 mysql mysql 98304 May 25 15:58 emp_2#P#p2.ibd
```

注意：
※约束条件、字符集等等也必须一致，建议使用show create table t1; 来获取创建表的SQL，否则在新服务器上导入表空间的时候会提示1808错误。

--：在目标数据库上，丢弃分区表的表空间

[root@localhost test]# ll  ---这时候在看，

```
MySQL [test]> alter table emp_2 discard tablespace;
Query OK, 0 rows affected (0.12 sec)
```

刚才的3个分区的idb文件都没有了

--：在源数据库上运行FLUS

```
-rw-r----- 1 mysql mysql 8604 May 25 04:14 emp_2.frm
```

H TABLES … FOR EXPORT 锁定表并生成.cfg元数据文件，最后将cfg和ibd文件传输到目标数据库中

```
mysql> flush tables emp_2 for export;
Query OK, 0 rows affected (0.00 sec)

[root@localhost zhangdb]# scp emp_2* root@192.168.2.100:/mysql/data/test/  --将文件cp到目标数据库

mysql> unlock tables;  ---最后将表的锁是否

--：在目标数据库中对文件授权，然后导入表空间查看数据是否完整可用
[root@localhost test]# chown mysql.mysql emp_2#*

MySQL [test]> alter table emp_2 import tablespace;
Query OK, 0 rows affected (0.96 sec)

MySQL [test]> select count(*) from emp_2;
+----------+
| count(*) |
+----------+
|   2999 |
+----------+
1 row in set (0.63 sec)
```

从上面的查看得知，分区表都已经导入到目标数据库中了，

另外，也可以将部分子分区导入到目标数据库中，（往往整个分区表会很大，可用只将需要用到的子分区导入到目标数据库中），
将部分子分区导入到目标数据库的方法是：
1）在创建目标表的时候，只需要创建要导入的分区即可，如： 只创建了p2 p3两个分区

```
 CREATE TABLE `emp_2` (
 `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
 `x` varchar(500) NOT NULL,
 `y` varchar(500) NOT NULL,
 PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=3000 DEFAULT CHARSET=utf8mb4
/*!50500 PARTITION BY RANGE COLUMNS(id)
(
 PARTITION p2 VALUES LESS THAN (2000) ENGINE = InnoDB,
 PARTITION p3 VALUES LESS THAN (3000) ENGINE = InnoDB) */
```

2）从源库cp到目标库的文件，当然也就是这俩的，就不需要其他分区的了，
3）其他的操作方法都一样了。

 

## 五、如何获取分区的相关信息



### 1. 通过 SHOW CREATE TABLE 语句来查看分区表的分区子句

譬如:

```
mysql> show create table e/G
```



### 2. 通过 SHOW TABLE STATUS 语句来查看表是否分区对应Create_options字段

譬如:

```
mysql> show table status/G

*************************** 1. row ***************************

Name: e Engine: InnoDB Version: 10 Row_format: Compact Rows: 6 Avg_row_length: 10922 Data_length: 65536Max_data_length: 0 Index_length: 0 Data_free: 0 Auto_increment: NULL Create_time: 2015-12-07 22:26:06 Update_time: NULL Check_time: NULL Collation: latin1_swedish_ci Checksum: NULL Create_options: partitioned Comment:
```



### 3. 查看 INFORMATION_SCHEMA.PARTITIONS表



### 4. 通过 EXPLAIN PARTITIONS SELECT 语句查看对于具体的SELECT语句,会访问哪个分区。





分区表创建实例

```
DROP TABLE IF EXISTS `t_receivedatainfo_copy`;
CREATE TABLE `t_receivedatainfo_copy` (
  `Id` bigint NOT NULL,
  `EquipmentCode` varchar(255) DEFAULT NULL,
  `Rfid` varchar(1000) DEFAULT NULL,
  `ReceiveData` longtext,
  `AddTime` datetime(3) NOT NULL,
  `MQName` varchar(50) DEFAULT NULL,
  PRIMARY KEY (`Id`,`AddTime`),
  KEY `receivedatainfo_Rfid` (`Rfid`) USING BTREE,
  KEY `receivedatainfo_EquipmentCode` (`EquipmentCode`) USING BTREE,
  KEY `receivedatainfo_AddTime` (`AddTime`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC
/*!50100 PARTITION BY RANGE (to_days(`AddTime`))
(PARTITION p0 VALUES LESS THAN (737790) ENGINE = InnoDB,
 PARTITION p1 VALUES LESS THAN (737821) ENGINE = InnoDB,
 PARTITION p2 VALUES LESS THAN (737850) ENGINE = InnoDB,
 PARTITION p3 VALUES LESS THAN (737881) ENGINE = InnoDB,
 PARTITION p4 VALUES LESS THAN (737911) ENGINE = InnoDB,
 PARTITION p5 VALUES LESS THAN (737942) ENGINE = InnoDB,
 PARTITION p6 VALUES LESS THAN (737972) ENGINE = InnoDB,
 PARTITION p7 VALUES LESS THAN (738003) ENGINE = InnoDB,
 PARTITION p8 VALUES LESS THAN (738034) ENGINE = InnoDB,
 PARTITION p9 VALUES LESS THAN (738064) ENGINE = InnoDB,
 PARTITION p10 VALUES LESS THAN (738095) ENGINE = InnoDB,
 PARTITION p11 VALUES LESS THAN (738125) ENGINE = InnoDB,
 PARTITION p12 VALUES LESS THAN MAXVALUE ENGINE = InnoDB) */;
```

