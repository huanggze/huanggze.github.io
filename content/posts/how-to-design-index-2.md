---
title: "如何设计 MySQL 索引（二）：实战"
date: 2023-01-14T20:42:52+08:00
toc: true
categories: ["mysql"]
---

## 题目一：联合索引

两种索引（status, create_time）、（create_time, status），以下 SQL 语句走哪个索引？[^1]

```sql
select  * from trade_info where status = 1 
    and create_time >= '2020-10-01 00:00:00'
    and create_time <= '2020-10-07 23:59:59';
```

执行 explain 语句，可以看到使用了索引（status, create_time）。

```shell
explain select * from trade_info where status = 1 and create_time >='2021-10-01 00:00:00' and create_time <= '2021-10-07 23:59:59'\G

*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: trade_info
   partitions: NULL
         type: index
possible_keys: idx_create_time_status,idx_status_create_time
          key: idx_create_time_status
      key_len: 9
          ref: NULL
         rows: 1
     filtered: 100.00
        Extra: Using where; Using index
```

进一步通过 optimizer_trace 来跟踪优化器的选择。MySQL 的 CBO（Cost-Based Optimizer 基于成本的优化器）选择的是复合索引 idx_status_create_time，因为该索引中的 status 和 create_time 都能参与了数据过滤，成本较低；而 idx_create_time_status 只有 create_time 参数数据过滤，status 被忽略了。

这是因为，status = 1 是精确匹配，create_time 是范围匹配，根据最左前缀匹配原则用（status，create_time）。

```sql
-- 开启optimizer_trace跟踪
set session optimizer_trace="enabled=on",end_markers_in_json=on;
-- 执行SQL语句
select * from trade_info where status = 1 and create_time >='2021-10-01 00:00:00' and create_time <= '2021-10-07 23:59:59';
-- 查看跟踪结果
SELECT trace FROM information_schema.OPTIMIZER_TRACE\G
```

```shell
*************************** 1. row ***************************
trace: {
  "steps": [
    // 省略...
    {
      "join_optimization": {
        "steps": [
          {
            "rows_estimation": [
              {
                "table": "`trade_info`",
                  "potential_range_indexes": [
                    {
                      "index": "PRIMARY",
                      "usable": false,
                      "cause": "not_applicable"
                    },
                    {
                      "index": "idx_create_time_status",
                      "usable": true,
                      "key_parts": [
                        "create_time",
                        "status",
                        "id"
                      ]
                    },
                    {
                      "index": "idx_status_create_time",
                      "usable": true,
                      "key_parts": [
                        "status",
                        "create_time",
                        "id"
                      ]
                    }
                  ],
                  "analyzing_range_alternatives": {
                    "range_scan_alternatives": [
                      {
                        "index": "idx_create_time_status",
                        "ranges": [
                          "0x41cb0f <= create_time <= 0x47cb0f"
                        ],
                        "cost": 0.36,
                        "chosen": false,
                        "cause": "cost"
                      },
                      {
                        "index": "idx_status_create_time",
                        "ranges": [
                          "1 <= status <= 1 AND 0x41cb0f <= create_time <= 0x47cb0f"
                        ],
                        "cost": 0.36,
                        "chosen": false,
                        "cause": "cost"
                      }
                    ]
                  }
                }
            ]
          }
        ]
      }
    }
  ]
}
```

## 题目二：联合索引

where 的顺序，会对索引选择有影响吗？答案是不会，以下 SQL 仍然选择索引 idx_create_time_status。查询引擎会自动优化为匹配联合索引的顺序。

```sql
select  * from trade_info
    where create_time >= '2020-10-01 00:00:00'
    and create_time <= '2020-10-07 23:59:59'
    and status = 1;
```

## 题目三：联合索引

user表有索引（name, age, birthday）。注意以下例子中第一个，和第三个，等同，只用到了复合索引中的第一列 name。当创建 (a,b,c) 复合索引时，想要索引生效的话，只能使用 a 和 ab、ac 和 abc 三种组合。

```sql
SELECT * from user WHERE name='Tom';
-- 可以命中索引，用到（name）

SELECT * from user WHERE name='Tom' AND age=18;
-- 可以命中索引，用到（name, age）

SELECT * from user WHERE name='Tom' AND birthday='2000-01-01 00:00:00'; 
-- 可以命中索引，等同于第一个，用到（name）

SELECT * from user WHERE name='Tom' AND age=18 AND birthday='2000-01-01 00:00:00'; 
-- 可以命中索引，用到（name, age, birthday）

SELECT * from user WHERE name='Tom' AND birthday='2000-01-01 00:00:00' AND age=18; 
-- 可以命中索引，用到（name, age, birthday）

SELECT * from user WHERE birthday='2000-01-01 00:00:00' AND age=18 AND name='Tom'; 
-- 可以命中索引，用到（name, age, birthday）

SELECT * from user WHERE age=18 AND birthday='2000-01-01 00:00:00'; 
-- 无法命中索引    
```

关于 EXPLAIN 语句的介绍：[http://khaidoan.wikidot.com/mysql-using-explain](http://khaidoan.wikidot.com/mysql-using-explain)。注意下面例子中的 key_len、ref 字段。

```shell
mysql> explain SELECT * from user WHERE name='Tom' AND birthday='2000-01-01 00:00:00';
+----+-------------+-------+------------+------+-------------------+-------------------+---------+-------+------+----------+-----------------------+
| id | select_type | table | partitions | type | possible_keys     | key               | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+-------+------------+------+-------------------+-------------------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | user  | NULL       | ref  | name_age_birthday | name_age_birthday | 123     | const |    1 |   100.00 | Using index condition |
+----+-------------+-------+------------+------+-------------------+-------------------+---------+-------+------+----------+-----------------------+

mysql> explain SELECT * from user WHERE birthday='2000-01-01 00:00:00' AND age=18 AND name='Tom';
+----+-------------+-------+------------+------+-------------------+-------------------+---------+-------------------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys     | key               | key_len | ref               | rows | filtered | Extra |
+----+-------------+-------+------------+------+-------------------+-------------------+---------+-------------------+------+----------+-------+
|  1 | SIMPLE      | user  | NULL       | ref  | name_age_birthday | name_age_birthday | 132     | const,const,const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+-------------------+-------------------+---------+-------------------+------+----------+-------+

mysql> explain SELECT * from user WHERE age=18 AND birthday='2000-01-01 00:00:00';
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | user  | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
```

## 题目四：排序查询

表 test 有联合索引（c1, c2, c3, c4）[^2]。下面这个c3及以后的索引全失效，因此只用到了两个索引，c3、c4要进行排序查找。

### 变体 1

```sql
SELECT * FROM test WHERE c1='a1' AND c2='a2' AND c4='a4' ORDER BY c3;
```

```shell
mysql> EXPLAIN SELECT * FROM test WHERE c1='a1' AND c2='a2' AND c4='a4' ORDER BY c3;
+----+-------------+-------+------------+------+---------------+-----------+---------+-------------+------+----------+-----------------------+
| id | select_type | table | partitions | type | possible_keys | key       | key_len | ref         | rows | filtered | Extra                 |
+----+-------------+-------+------------+------+---------------+-----------+---------+-------------+------+----------+-----------------------+
|  1 | SIMPLE      | test  | NULL       | ref  | idx_c1234     | idx_c1234 | 10      | const,const |    1 |   100.00 | Using index condition |
+----+-------------+-------+------------+------+---------------+-----------+---------+-------------+------+----------+-----------------------+
```

Extra 的解释表是怎么被处理的，一些值表明 query 是高效的（using index），一些表示是低效（using filesort、using temporary）。这里 using index condition 表示使用了索引下推（index condition push-down）。

### 变体 2

```sql
SELECT * FROM test WHERE c1='a1' AND c2='a2' ORDER BY c4;
```

用到了 c1,c2 索引，但是 c4 是用于排序，中间的 c3 索引断了，MySQL 会使用文件内排序给出查询结果，这样子就导致了性能下降。

```shell
mysql> EXPLAIN SELECT * FROM test WHERE c1='a1' AND c2='a2' ORDER BY c4;
+----+-------------+-------+------------+------+---------------+-----------+---------+-------------+------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key       | key_len | ref         | rows | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+-----------+---------+-------------+------+----------+----------------+
|  1 | SIMPLE      | test  | NULL       | ref  | idx_c1234     | idx_c1234 | 10      | const,const |    1 |   100.00 | Using filesort |
+----+-------------+-------+------------+------+---------------+-----------+---------+-------------+------+----------+----------------+
```

### 变体 3

```sql
SELECT * FROM test WHERE c1='a1' AND c2='a2' AND c5='a5' ORDER BY c2, c3;
```

用到了 c1，c2 索引，但是 c2、c3 是用于排序，没有出现 filesort，性能可以。

```shell
mysql> EXPLAIN SELECT * FROM test WHERE c1='a1' AND c2='a2' AND c5='a5' ORDER BY c2, c3;
+----+-------------+-------+------------+------+---------------+-----------+---------+-------------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key       | key_len | ref         | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+-----------+---------+-------------+------+----------+-------------+
|  1 | SIMPLE      | test  | NULL       | ref  | idx_c1234     | idx_c1234 | 10      | const,const |    1 |   100.00 | Using where |
+----+-------------+-------+------------+------+---------------+-----------+---------+-------------+------+----------+-------------+
```

### 变体 4

```sql
SELECT * FROM test WHERE c1='a1' AND c2='a2' AND c5='a5' ORDER BY c3, c2;
```

将c2、c3的顺序倒置，竟然没有出现filesort，这是因为c2=‘a2’，order by中的 c2 字段已经是一个常量了，所以真正排序的字段就只有c3.

```shell
mysql> EXPLAIN SELECT * FROM test WHERE c1='a1' AND c2='a2' AND c5='a5' ORDER BY c3, c2;
+----+-------------+-------+------------+------+---------------+-----------+---------+-------------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key       | key_len | ref         | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+-----------+---------+-------------+------+----------+-------------+
|  1 | SIMPLE      | test  | NULL       | ref  | idx_c1234     | idx_c1234 | 10      | const,const |    1 |   100.00 | Using where |
+----+-------------+-------+------------+------+---------------+-----------+---------+-------------+------+----------+-------------+
```

## 题目五：分组查询

group by表面上是分组，但实际上分组的前提必须排序。

```sql
SELECT c1,c2,c3,c4 FROM test WHERE c1='a1' AND c4='a4' GROUP BY c3, c2;
```

```shell
mysql> EXPLAIN SELECT c1,c2,c3,c4 FROM test WHERE c1='a1' AND c4='a4' GROUP BY c3, c2;
+----+-------------+-------+------------+------+---------------+-----------+---------+-------+------+----------+-------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key       | key_len | ref   | rows | filtered | Extra                                     |
+----+-------------+-------+------------+------+---------------+-----------+---------+-------+------+----------+-------------------------------------------+
|  1 | SIMPLE      | test  | NULL       | ref  | idx_c1234     | idx_c1234 | 5       | const |    1 |   100.00 | Using where; Using index; Using temporary |
+----+-------------+-------+------------+------+---------------+-----------+---------+-------+------+----------+-------------------------------------------+
```

## 题目六：分页查询

```sql
SELECT  * FROM trade_info WHERE status = 0 
    AND create_time >= '2020-10-01 00:00:00'
    AND create_time <= '2020-10-07 23:59:59'
    ORDER BY id DESC LIMIT 102120, 20;
```

对于典型的分页 limit m, n 来说，越往后翻页越慢，也就是m越大会越慢，因为要定位m位置需要扫描的数据越来越多，导致IO开销比较大。

```shell
mysql> EXPLAIN SELECT * FROM trade_info WHERE status = 0 AND create_time >= '2020-10-01 00:00:00' AND create_time <= '2020-10-07 23:59:59' ORDER BY id DESC LIMIT 102120, 20\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: trade_info
   partitions: NULL
         type: index
possible_keys: create_time_status,status_create_time
          key: PRIMARY
      key_len: 4
          ref: NULL
         rows: 1
     filtered: 100.00
        Extra: Using where; Backward index scan
```

这里可以利用辅助索引的【索引覆盖】来进行优化，先获取 id，这一步就是索引覆盖扫描，不需要回表，然后通过 id 跟原表 trade_info 进行关联，改写后的 SQL 如下：

```sql
SELECT * FROM trade_info AS a,
    (SELECT id FROM trade_info WHERE status = 0
        AND create_time >= '2020-10-01 00:00:00'
        AND create_time <= '2020-10-07 23:59:59'
        ORDER BY id DESC LIMIT 102120, 20
    ) AS b -- 这一步走的是索引覆盖扫描，不需要回表
WHERE a.id = b.id;
```

```shell
mysql> explain select * from trade_info as a, (select  id from trade_info where status = 0 and create_time >= '2020-10-01 00:00:00' and create_time <= '2020-10-07 23:59:59' order by id desc limit 102120, 20) as b where a.id = b.id\G
*************************** 1. row ***************************
           id: 1
  select_type: PRIMARY
        table: a
   partitions: NULL
         type: index
possible_keys: PRIMARY
          key: idx_status_create_time
      key_len: 9
          ref: NULL
         rows: 1
     filtered: 100.00
        Extra: Using index
*************************** 2. row ***************************
           id: 1
  select_type: PRIMARY
        table: <derived2>
   partitions: NULL
         type: ref
possible_keys: <auto_key0>
          key: <auto_key0>
      key_len: 4
          ref: test.a.id
         rows: 2
     filtered: 100.00
        Extra: Using index
*************************** 3. row ***************************
           id: 2
  select_type: DERIVED
        table: trade_info
   partitions: NULL
         type: index
possible_keys: idx_status_create_time,idx_create_time_status
          key: PRIMARY
      key_len: 4
          ref: NULL
         rows: 1
     filtered: 100.00
        Extra: Using where; Backward index scan
```

## 题目七：批量更新大数据

营销系统有一批过期的优惠卷要失效，核心SQL如下：

```sql
-- 需要更新的数据量500w
update coupons set status = 1 where status =0 and create_time >= '2020-10-01 00:00:00' and create_time <= '2020-10-07 23:59:59';
```

在 Oracle 里更新 500w 数据是很快，因为可以利用多个 cpu core 去执行，但是 MySQL 就需要注意了，一个 SQL 只能使用一个 cpu core 去处理，如果SQL很复杂或执行很慢，就会阻塞后面的 SQL 请求，造成活动连接数暴增，MySQL CPU 100%，相应的接口 Timeout，同时对于主从复制架构，而且做了业务读写分离，更新 500w 数据需要 5 分钟，Master 上执行了 5 分钟，binlog 传到了 Slave 也需要执行 5 分钟，那就是 Slave 延迟 5 分钟，在这期间会造成业务脏数据，比如重复下单等[^1]。

优化思路：分而治之。先获取 where 条件中的最小 id 和最大 id，然后分批次去更新，每个批次 1000 条，这样既能快速完成更新，又能保证主从复制不会出现延迟。

1. 先获取要更新的数据范围内的最小 id 和最大 id（表没有物理 delete，所以 id 是连续的）

```shell
mysql> explain select min(id) min_id, max(id) max_id from coupons where status =0 and create_time >= '2020-10-01 00:00:00' and create_time <= '2020-10-07 23:59:59'; 
+----+-------------+-------+------------+-------+------------------------+------------------------+---------+---
| id | select_type | table | partitions | type  | possible_keys          | key                    | key_len | ref  | rows   | filtered | Extra                    |
+----+-------------+-------+------------+-------+------------------------+------------------------+---------+---
|  1 | SIMPLE      | users | NULL       | range | idx_status_create_time | idx_status_create_time | 6       | NULL | 180300 |   100.00 | Using where; Using index |
```

2. 以每次 1000 条 commit 一次进行循环 update，主要代码如下：

```shell
current_id = min_id;
for  current_id < max_id do
  update coupons set status = 1 where id >=current_id and id <= current_id + 1000;  // 通过主键id更新 1000 条很快
commit;
current_id += 1000;
done
```

## 附录

```sql
-- 题目一
CREATE TABLE trade_info (
    id INT,
    status INT,
    create_time DATE,
    PRIMARY KEY (id),
    KEY idx_status_create_time (status, create_time),
    KEY idx_create_time_status (create_time, status)
);

-- 删除索引
DROP INDEX idx_status_create_time ON trade_info;

-- 创建索引
CREATE INDEX idx_status_create_time ON trade_info(status, create_time);


-- 题目三
CREATE TABLE user (
    id INT,
    name VARCHAR(30),
    age INT,
    birthday DATE,
    addr VARCHAR(30), -- 必须要多加几个其他字段做测试
    PRIMARY KEY (id),
    KEY name_age_birthday (name, age, birthday)
);

-- 题目四
CREATE TABLE test (
  id INT,
  c1 INT,
  c2 INT,
  c3 INT,
  c4 INT,
  c5 INT,
  PRIMARY KEY (id),
  KEY idx_c1234 (c1, c2, c3, c4)
);
```

## 参考资料

[^1] [阿里面试官：MySQL如何设计索引更高效？](https://segmentfault.com/a/1190000038921156)
[^2] [MySQL索引面试题分析](https://developer.aliyun.com/article/1113711)