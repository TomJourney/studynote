# mysql8 index hint（索引提示）

【readme】

本文总结自 https://dev.mysql.com/doc/refman/8.0/en/index-hints.html 



# 【1】索引提示

1. 索引提示给了优化器关于在查询执行时如何选择索引的信息； 
2. 本文描述的索引提示，不同于 优化器提示（https://dev.mysql.com/doc/refman/8.0/en/optimizer-hints.html）。 索引提示与优化器提示可以单独，也可以一起使用； 

2. 索引提示可以应用到 select 和 update语句，也可以应用到多表删除delete语句，但不能用到单表删除delete语句；
3. 索引提示放在表名后面；（关于在SELECT语句中指定表的通用语法，参见 https://dev.mysql.com/doc/refman/8.0/en/join.html） 

4. 单个表的索引提示语法如下：

   1. ```sql
      tbl_name [[AS] alias] [index_hint_list]
      
      index_hint_list:
          index_hint [index_hint] ...
      
      index_hint:
          USE {INDEX|KEY}
            [FOR {JOIN|ORDER BY|GROUP BY}] ([index_list])
        | {IGNORE|FORCE} {INDEX|KEY}
            [FOR {JOIN|ORDER BY|GROUP BY}] (index_list)
      
      index_list:
          index_name [, index_name] ...
      ```

   

## 【1.1】索引提示语法说明

USE INDEX 提示：告诉mysql 只能使用其中一个索引查找数据； 

IGNORE INDEX 提示：告诉mysql 不要使用给定索引；

FORCE INDEX 提示：类似与 USE INDEX，此外还要假定表扫描的开销非常大；即，  仅当无法使用给定索引之一查找数据时，才使用表扫描； 

```c++
【提示】
从 MySQL 8.0.20 开始，服务器支持索引级优化器提示 JOIN_INDEX、GROUP_INDEX、ORDER_INDEX 和 INDEX（它们相当于并旨在取代 FORCE INDEX 索引提示）以及 NO_JOIN_INDEX、NO_GROUP_INDEX、NO_ORDER_INDEX 和 NO_INDEX 优化器提示，相当于并旨在取代 IGNORE INDEX 索引提示。 因此，您应该预期 USE INDEX、FORCE INDEX 和 IGNORE INDEX 在 MySQL 的未来版本中将被弃用，并在之后的某个时间被完全删除。
单表和多表 DELETE 语句都支持这些索引级优化器提示。
更多信息，参考https://dev.mysql.com/doc/refman/8.0/en/optimizer-hints.html#optimizer-hints-index-level
```

【例】 使用 索引提示的例子：

```sql
-- 使用 col1_index， col2_index 索引
SELECT * FROM table1 USE INDEX (col1_index,col2_index)
  WHERE col1=1 AND col2=2 AND col3=3;

SELECT * FROM table1 IGNORE INDEX (col3_index)
  WHERE col1=1 AND col2=2 AND col3=3;
```



### 【1.1.1】索引提示语法的特征

1. 在 USE INDEX 中省略 index_list 在语法上是有效的，这意味着“不使用索引”。 省略 FORCE INDEX 或 IGNORE INDEX 的 index_list 是一个语法错误。

2. 你可以在索引提示中添加 FOR子句，来指定索引提示的范围； 这为查询处理的不同阶段的执行计划的优化器选择提供了更加精细化的控制；

   1. 仅影响mysql查找数据及处理连表join时使用的索引，使用 FOR JOIN；
   2. 要影响排序或分组的索引使用，使用 FOR ORDER 或 FOR  GROUP BY ； 

3. 你可以指定多个索引提示：

   1. ```sql
      SELECT * FROM t1 USE INDEX (i1) IGNORE INDEX FOR ORDER BY (i2) ORDER BY a;
      ```

4. 如果索引提示不包含 FOR 子句， 则该提示会作用到语句的所有部分；

   1. 如 IGNORE INDEX (i1) ， 它等价于如下提示：

   2. ```sql
      IGNORE INDEX FOR JOIN (i1)
      IGNORE INDEX FOR ORDER BY (i1)
      IGNORE INDEX FOR GROUP BY (i1)
      ```

## 【1.2】索引提示应用的顺序 

1. 索引提示将按照如下顺序进行应用：

   1. (USE | FORCE) INDEX 如果有就应用； （如果没有，则优化器决定使用的索引集合）

   2. IGNORE INDEX 应用到上一步的结果； 如， 以下2个查询语句等价：

      1. ```sql
         SELECT * FROM t1 USE INDEX (i1) IGNORE INDEX (i2) USE INDEX (i2);
         
         SELECT * FROM t1 USE INDEX (i1);
         ```

2. 对于全文搜索， 索引提示的工作方式如下：

   1. 对于自然语言模式搜索： 索引提示会被默认忽略。 举个例子， IGNORE INDEX(i1)  会被忽略且没有提示，该索引可以继续使用；

   2. 对于布尔模式搜索：  

      1. FOR ORDER BY 或 FOR GROUP BY 的索引提示将被忽略；

      2. FOR JOIN 或 没有FOR子句的索引提示将会被尊重； 

      3. 与 索引提示应用到非全文搜索的方式相反， 全文搜索的提示应用到查询执行的所有阶段（查找数据行，检索，分组，排序）； 即使针对非全文索索给出的提示也是如此；

      4. 例子： 以下2个查询语句等价：

         1. ```sql
            SELECT * FROM t
              USE INDEX (index1)
              IGNORE INDEX FOR ORDER BY (index1)
              IGNORE INDEX FOR GROUP BY (index1)
              WHERE ... IN BOOLEAN MODE ... ;
            
            SELECT * FROM t
              USE INDEX (index1)
              WHERE ... IN BOOLEAN MODE ... ;
            ```



## 【1.3】DELETE 语句的索引提示

索引提示能够应用到 DELETE语句， 当且仅当使用多个表的DELETE 语法，如下所示。 

```sql
mysql> EXPLAIN DELETE FROM t1 USE INDEX(col2) 
    -> WHERE col1 BETWEEN 1 AND 100 AND COL2 BETWEEN 1 AND 100\G
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that
corresponds to your MySQL server version for the right syntax to use near 'use
index(col2) where col1 between 1 and 100 and col2 between 1 and 100' at line 1
  
mysql> EXPLAIN DELETE t1.* FROM t1 USE INDEX(col2) 
    -> WHERE col1 BETWEEN 1 AND 100 AND COL2 BETWEEN 1 AND 100\G
*************************** 1. row ***************************
           id: 1
  select_type: DELETE
        table: t1
   partitions: NULL
         type: range
possible_keys: col2
          key: col2
      key_len: 5
          ref: NULL
         rows: 72
     filtered: 11.11
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
```

