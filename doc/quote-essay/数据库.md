### 数据库mysql使用JSON 

#### JSON 数据类型

JSON（JavaScript Object Notation）主要用于互联网应用服务之间的数据交换。MySQL 支持RFC 7159定义的 JSON 规范，主要有 `JSON 对象` 和 `JSON 数组`两种类型。

> 需要注意是，JSON 类型是从 MySQL 5.7 版本开始支持的功能，而 8.0 版本解决了更新 JSON 的日志性能瓶颈。如果要在生产环境中使用 JSON 数据类型，强烈推荐使用 MySQL 8.0 版本。

#### 业务表结构设计实战

**用户登录设计**

在数据库中，JSON 类型比较适合存储一些修改较少、相对静态的数据，比如用户登录信息的存储如下：

```sql
DROP TABLE IF EXISTS UserLogin;

CREATE TABLE UserLogin (
    userId BIGINT NOT NULL,
    loginInfo JSON,
    PRIMARY KEY(userId)
);
```

由于当前业务的登录方式越来越多样化，如同一账户支持手机、微信、QQ 账号登录，所以这里可以用 JSON 类型存储登录的信息。

接着，插入下面的数据：

```sql
SET @a = '
{
   "cellphone" : "1",
   "wxchat" : "码农",
   "77" : "1"
}
';
INSERT INTO UserLogin VALUES (1,@a);

SET @b = '
{  
  "cellphone" : "1188"
}
';

INSERT INTO UserLogin VALUES (2,@b);
```

从上面的例子中可以看到，用户 1 登录有三种方式：手机验证码登录、微信登录、QQ 登录，而用户 2 只有手机验证码登录。

而如果不采用 JSON 数据类型，就要用下面的方式建表：

```sql
SELECT
    userId,
    JSON_UNQUOTE(JSON_EXTRACT(loginInfo,"$.cellphone")) cellphone,
    JSON_UNQUOTE(JSON_EXTRACT(loginInfo,"$.wxchat")) wxchat
FROM UserLogin;
```

当然了，每次写 `JSON_EXTRACT`、`JSON_UNQUOTE` 非常麻烦，MySQL 还提供了 ->> 表达式，和上述 SQL 效果完全一样：

```sql
SELECT 
    userId,
    loginInfo->>"$.cellphone" cellphone,
    loginInfo->>"$.wxchat" wxchat
FROM UserLogin;
```

当 JSON 数据量非常大，用户希望对 JSON 数据进行有效检索时，可以利用 MySQL 的 函数索引 功能对 JSON 中的某个字段进行索引。

比如在上面的用户登录示例中，假设用户必须绑定唯一手机号，且希望未来能用手机号码进行用户检索时，可以创建下面的索引：

```sql
ALTER TABLE UserLogin ADD COLUMN cellphone VARCHAR(255) AS (loginInfo->>"$.cellphone");

ALTER TABLE UserLogin ADD UNIQUE INDEX idx_cellphone(cellphone);
```

上述 SQL 首先创建了一个虚拟列 cellphone，这个列是由函数 `loginInfo->>"$.cellphone"` 计算得到的。然后在这个虚拟列上创建一个唯一索引 `idx_cellphone`。这时再通过虚拟列 cellphone 进行查询，就可以看到优化器会使用到新创建的 `idx_cellphone` 索引：

```sql
EXPLAIN SELECT  *  FROM UserLogin WHERE cellphone = '1'


*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: UserLogin
   partitions: NULL
         type: const
possible_keys: idx_cellphone
          key: idx_cellphone
      key_len: 1023
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```

当然，我们可以在一开始创建表的时候，就完成虚拟列及函数索引的创建。如下表创建的列 cellphone 对应的就是 JSON 中的内容，是个虚拟列；`uk_idx_cellphone` 就是在虚拟列 cellphone 上所创建的索引

```sql
CREATE TABLE UserLogin (
    userId BIGINT,
    loginInfo JSON,
    cellphone VARCHAR(255) AS (loginInfo->>"$.cellphone"),
    PRIMARY KEY(userId),
    UNIQUE KEY uk_idx_cellphone(cellphone)
);
```

**用户画像设计**

某些业务需要做用户画像（也就是对用户打标签），然后根据用户的标签，通过数据挖掘技术，进行相应的产品推荐。比如：

- 在电商行业中，根据用户的穿搭喜好，推荐相应的商品；
- 在音乐行业中，根据用户喜欢的音乐风格和常听的歌手，推荐相应的歌曲；
- 在金融行业，根据用户的风险喜好和投资经验，推荐相应的理财产品。

在这，我强烈推荐你用 JSON 类型在数据库中存储用户画像信息，并结合 JSON 数组类型和多值索引的特点进行高效查询。假设有张画像定义表：

```sql
CREATE TABLE Tags (
    tagId bigint auto_increment,
    tagName varchar(255) NOT NULL,
    primary key(tagId)
);

SELECT * FROM Tags;
+-------+--------------+
| tagId | tagName      |
+-------+--------------+
|     1 | 70后         |
|     2 | 80后         |
|     3 | 90后         |
|     4 | 00后         |
|     5 | 爱运动       |
|     6 | 高学历       |
|     7 | 小资         |
|     8 | 有房         |
|     9 | 有车         |
|    10 | 常看电影     |
|    11 | 爱网购       |
|    12 | 爱外卖       |
+-------+--------------+
```

可以看到，表 Tags 是一张画像定义表，用于描述当前定义有多少个标签，接着给每个用户打标签，比如用户 David，他的标签是 80 后、高学历、小资、有房、常看电影；用户 Tom，90 后、常看电影、爱外卖。

若不用 JSON 数据类型进行标签存储，通常会将用户标签通过字符串，加上分割符的方式，在一个字段中存取用户所有的标签：

```sql
+-------+---------------------------------------+
|用户    |标签                                   |
+-------+---------------------------------------+
|David  |80后 ； 高学历 ； 小资 ； 有房 ；常看电影   |
|Tom    |90后 ；常看电影 ； 爱外卖                 |
+-------+---------------------------------------
```

这样做的缺点是：不好搜索特定画像的用户，另外分隔符也是一种自我约定，在数据库中其实可以任意存储其他数据，最终产生脏数据。

用 JSON 数据类型就能很好解决这个问题：

```sql
DROP TABLE IF EXISTS UserTag;
CREATE TABLE UserTag (
    userId bigint NOT NULL,
    userTags JSON,
    PRIMARY KEY (userId)
);

INSERT INTO UserTag VALUES (1,'[2,6,8,10]');
INSERT INTO UserTag VALUES (2,'[3,10,12]');
```

其中，userTags 存储的标签就是表 Tags 已定义的那些标签值，只是使用 JSON 数组类型进行存储。

MySQL 8.0.17 版本开始支持 `Multi-Valued Indexes`，用于在 JSON 数组上创建索引，并通过函数 `member of`、`json_contains`、`json_overlaps` 来快速检索索引数据。所以你可以在表 UserTag 上创建 `Multi-Valued Indexes`：

```sql
ALTER TABLE UserTag ADD INDEX idx_user_tags ((cast((userTags->"$") as unsigned array)));
```

如果想要查询用户画像为常看电影的用户，可以使用函数 MEMBER OF：

```sql
EXPLAIN SELECT * FROM UserTag WHERE 10 MEMBER OF(userTags->"$")

*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: UserTag
   partitions: NULL
         type: ref
possible_keys: idx_user_tags
          key: idx_user_tags
      key_len: 9
          ref: const
         rows: 1
     filtered: 100.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)


SELECT * FROM UserTag WHERE 10 MEMBER OF(userTags->"$");

+--------+---------------+
| userId | userTags      |
+--------+---------------+
|      1 | [2, 6, 8, 10] |
|      2 | [3, 10, 12]   |
+--------+---------------+
2 rows in set (0.00 sec)
```

如果想要查询画像为 80 后，且常看电影的用户，可以使用函数 `JSON_CONTAINS`：

```sql
EXPLAIN SELECT * FROM UserTag WHERE JSON_CONTAINS(userTags->"$", '[2,10]')

*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: UserTag
   partitions: NULL
         type: range
possible_keys: idx_user_tags
          key: idx_user_tags
      key_len: 9
          ref: NULL
         rows: 3
     filtered: 100.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)

SELECT * FROM UserTag WHERE JSON_CONTAINS(userTags->"$", '[2,10]');

+--------+---------------+
| userId | userTags      |
+--------+---------------+
|      1 | [2, 6, 8, 10] |
+--------+---------------+
1 row in set (0.00 sec)
```

如果想要查询画像为 80 后、90 后，且常看电影的用户，则可以使用函数 `JSON_OVERLAP`：

```sql
EXPLAIN SELECT * FROM UserTag WHERE JSON_OVERLAPS(userTags->"$", '[2,3,10]')

*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: UserTag
   partitions: NULL
         type: range
possible_keys: idx_user_tags
          key: idx_user_tags
      key_len: 9
          ref: NULL
         rows: 4
     filtered: 100.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)

SELECT * FROM UserTag WHERE JSON_OVERLAPS(userTags->"$", '[2,3,10]');

+--------+---------------+
| userId | userTags      |
+--------+---------------+
|      1 | [2, 6, 8, 10] |
|      2 | [3, 10, 12]   |
+--------+---------------+
2 rows in set (0.01 sec)
```

JSON 类型是 MySQL 5.7 版本新增的数据类型，用好 JSON 数据类型可以有效解决很多业务中实际问题。最后，我总结下今天的重点内容：

- 使用 JSON 数据类型，推荐用 MySQL 8.0.17 以上的版本，性能更好，同时也支持 `Multi-Valued Indexes`；
- JSON 数据类型的好处是无须预先定义列，数据本身就具有很好的描述性；
- 不要将有明显关系型的数据用 JSON 存储，如用户余额、用户姓名、用户身份证等，这些都是每个用户必须包含的数据；
- JSON 数据类型推荐使用在不经常更新的静态数据存储。

### SQL写法 :行行比较

- 环境准备
- 需求背景
- 循环查询
- OR 拼接
- 混查过滤
- 行行比较
- 总结

#### 环境准备

数据库版本：MySQL 5.7.20-log

建表 SQL

```sql
DROP TABLE IF EXISTS `t_ware_sale_statistics`;
CREATE TABLE `t_ware_sale_statistics` (  
`id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键id',  
`business_id` bigint(20) NOT NULL COMMENT '业务机构编码', 
`ware_inside_code` bigint(20) NOT NULL COMMENT '商品自编码',  
`weight_sale_cnt_day` double(16,4) DEFAULT NULL COMMENT '平均日销量',  
`last_thirty_days_sales` double(16,4) DEFAULT NULL COMMENT '最近30天销量',  
`last_sixty_days_sales` double(16,4) DEFAULT NULL COMMENT '最近60天销量',  
`last_ninety_days_sales` double(16,4) DEFAULT NULL COMMENT '最近90天销量',  
`same_period_sale_qty_thirty` double(16,4) DEFAULT NULL COMMENT '去年同期30天销量',  `same_period_sale_qty_sixty` double(16,4) DEFAULT NULL COMMENT '去年同期60天销量',  `same_period_sale_qty_ninety` double(16,4) DEFAULT NULL COMMENT '去年同期90天销量',  
`create_user` bigint(20) DEFAULT NULL COMMENT '创建人',  
`create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',  
`modify_user` bigint(20) DEFAULT NULL COMMENT '最终修改人',  
`modify_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '最终修改时间',  `is_delete` tinyint(2) DEFAULT '2' COMMENT '是否删除，1：是，2：否', 
PRIMARY KEY (`id`) USING BTREE,  
KEY `idx_business_ware` (`business_id`,`ware_inside_code`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 ROW_FORMAT=DYNAMIC COMMENT='商品销售统计';
```

初始化数据

准备了 769063 条数据

#### 需求背景

业务机构下销售商品，同个业务机构可以销售不同的商品，同个商品可以在不同的业务机构销售，也就说：业务机构与商品是多对多的关系

假设现在有 n 个机构，每个机构下有几个商品，如何查询出这几个门店下各自商品的销售情况？

具体点，类似如下

![图片](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupXibswLiciaPvFKl4dZW663icMLP57aicNt2gSxlDRGsN3HXfxI6qnNUFrJhic6sSAoJuccsvjOGvyNib6WA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)图片

如何查出 100001 下商品 1000、1001、1003 、 100002 下商品 1003、1004 、 100003 下商品 1006、1008、1009 的销售情况

相当于是双层列表（业务机构列表中套商品列表）的查询；业务机构列表和商品列表都不是固定的，而是动态的

那么问题就是：如何查询多个业务机构下，某些商品的销售情况

（问题经我一描述，可能更模糊了，大家明白意思了就好！）

#### 循环查询

这个很容易想到，在代码层面循环业务机构列表，每个业务机构查一次数据库，伪代码如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupXibswLiciaPvFKl4dZW663icMLxYo8LzG9MHicdHhI4QO6CHlfuxhBS7Xs7qFoqUMb4mzPHNwHkwOacaQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

具体的 SQL 类似如下

![图片](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupXibswLiciaPvFKl4dZW663icMLFszTCzn8vDcEPBBYVBrLWyVar7hh9FMibsC3CJwtbL1icN8pJPDjoKFg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

SQL 能走索引

![图片](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupXibswLiciaPvFKl4dZW663icML7qT0npFmlapgyOjud5NnHUSESoev5hKNsWocjt58Y0UGJSTfIc0dWA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

实现简单，也好理解，SQL 也能走索引，一切看起来似乎很完美

然而现实是：部门开发规范约束，不能循环查数据库

哦豁，这种方式只能放弃，另寻其他方式了

#### OR 拼接

通过 MyBatis 的 动态 SQL 功能，进行 SQL 拼接，类似如下

![图片](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupXibswLiciaPvFKl4dZW663icMLxTA8Iib2FQ3AVOg9nibnmMG4uYkfypF8kUpPVSWEXicewk6N2xicQmq7SQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

具体的 SQL 类似如下

![图片](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupXibswLiciaPvFKl4dZW663icMLI5S9l3GTsz3d5fUK4V0rsqv84RVMWhbw0vYIjI9vzLkvibKIuIgrRtw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

SQL 也能走索引

![图片](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupXibswLiciaPvFKl4dZW663icMLacL3333oO4pokibRVjNoWovUZyTmwKgFUsnr8FIeIlwCFrCNFo6eN8w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

实现简单，也好理解，SQL 也能走索引，而且只查询一次数据库，貌似可行

唯一可惜的是：有点费 OR，如果业务机构比较多，那 SQL 会比较长

作为候选人之一吧，我们接着往下看

#### 混查过滤

同样是利用 Mybatis 的 动态 SQL ，将 business_id 列表拼在一起、 ware_inside_code 拼在一起，类似如下

![图片](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupXibswLiciaPvFKl4dZW663icMLLcjs4oKRTSAGuyxerBPGaPRBVGfiadcw3GDFUOLfW0JJMDuPoQIPyHg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

具体的 SQL 类似如下

![图片](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupXibswLiciaPvFKl4dZW663icMLn5NzsOAEmVouFWdib6ib0b1sb1RCf1ey5sh95qS7sc3vLI60hVjmKnTA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

SQL 也能走索引

![图片](https://mmbiz.qpic.cn/mmbiz_png/6mychickmupXibswLiciaPvFKl4dZW663icMLHQboz58BmG7vEMKqW6xGgRQQlOlJWvTUl9FhibOoDm1YOsvd22kYvgA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

实现简单，也好理解，SQL 也能走索引，而且只查询一次数据库，似乎可行

但是：查出来的结果集大于等于我们想要的结果集，你品，你细品！

所以还需要对查出来的结果集进行一次过滤，过滤出我们想要的结果集

姑且也作为候选人之一吧，我们继续往下看

#### 行行比较

SQL-92 中加入了行与行比较的功能，这样一来，比较谓词 = 、< 、> 和 IN 谓词的参数就不再只是标量值了，还可以是值列表了

当然，还是得用到 Mybatis 的 动态 SQL ，类似如下

```sql
SELECT * FROM t_ware_sale_statistics 
WHERE (business_id,ware_inside_code) IN (
<foreach collection="businessWareList" item="businessWare" separator=","> 
 (#{item.businessId},#{item.wareInsideCode})
</foreach>    
)
```

具体的 SQL 类似如下

```sql
SELECT * FROM t_ware_sale_statistics 
WHERE (business_id,ware_inside_code) 
IN ((100120609,5169),(100120609,138),(100120609,14418));
```

SQL 同样能走索引（`range`）

实现简单，SQL 也能走索引，而且只查询一次数据库，感觉可行

另外，行行比较是 SQL 规范，不是某个关系型数据库的规范，也就说关系型数据库都应该支持这种写法

#### 总结

2、某一个需求的实现往往有很多种方式，我们需要结合业务以及各种约束综合考虑，选择最合适的那个

3、行行比较是 SQL-92 中引入的，SQL-92 是 1992 年制定的规范

行行比较不是新特性，而是很早就存在的基础功能！

### 5大步骤+10个案例，SQL优化方法

#### 一、前言

在应用开发的早期，数据量少，开发人员开发功能时更重视功能上的实现，随着生产数据的增长，很多SQL语句开始暴露出性能问题，对生产的影响也越来越大，有时可能这些有问题的SQL就是整个系统性能的瓶颈。

#### 二、SQL优化一般步骤

**1、通过慢查日志等定位那些执行效率较低的SQL语句**

**2、explain 分析SQL的执行计划**

需要重点关注`type、rows、filtered、extra`。

type由上至下，效率越来越高。

- **ALL** 全表扫描；
- **index** 索引全扫描；
- **range** 索引范围扫描，常用语<,<=,>=,between,in等操作；
- **ref** 使用非唯一索引扫描或唯一索引前缀扫描，返回单条记录，常出现在关联查询中；
- **eq_ref** 类似ref，区别在于使用的是唯一索引，使用主键的关联查询；
- **const/system** 单条记录，系统会把匹配行中的其他列作为常数处理，如主键或唯一索引查询；
- **null MySQL**不访问任何表或索引，直接返回结果；
- 虽然上至下，效率越来越高，但是根据cost模型，假设有两个索引idx1(a, b, c),idx2(a, c)，SQL为"select * from t where a = 1 and b in (1, 2) order by c";如果走idx1，那么是type为range，如果走idx2，那么type是ref；当需要扫描的行数，使用idx2大约是idx1的5倍以上时，会用idx1，否则会用idx2。

**Extra**

- **Using filesort：**MySQL需要额外的一次传递，以找出如何按排序顺序检索行。通过根据联接类型浏览所有行并为所有匹配WHERE子句的行保存排序关键字和行的指针来完成排序。然后关键字被排序，并按排序顺序检索行；
- **Using temporary：**使用了临时表保存中间结果，性能特别差，需要重点优化；
- **Using index：**表示相应的 select 操作中使用了覆盖索引（Coveing Index）,避免访问了表的数据行，效率不错！如果同时出现 using where，意味着无法直接通过索引查找来查询到符合条件的数据；
- **Using index condition：**MySQL5.6之后新增的ICP，using index condtion就是使用了ICP（索引下推），在存储引擎层进行数据过滤，而不是在服务层过滤，利用索引现有的数据减少回表的数据。

**3、show profile 分析**

了解SQL执行的线程的状态及消耗的时间。

默认是关闭的，开启语句“set profiling = 1;”

```sql
SHOW PROFILES ;
SHOW PROFILE FOR QUERY  #{id};
```

**4、trace**

trace分析优化器如何选择执行计划，通过trace文件能够进一步了解为什么优惠券选择A执行计划而不选择B执行计划。

```sql
set optimizer_trace="enabled=on";
set optimizer_trace_max_mem_size=1000000;
select * from information_schema.optimizer_trace;
```

**5、确定问题并采用相应的措施**

- 优化索引；
- 优化SQL语句：修改SQL、IN 查询分段、时间查询分段、基于上一次数据过滤；
- 改用其他实现方式：ES、数仓等；
- 数据碎片处理。

#### 三、场景分析

**1、最左匹配**

**1）索引**

```sql
KEY `idx_shopid_orderno` (`shop_id`,`order_no`)
```

**2）SQL语句**

```sql
select * from _t where orderno=''
```

查询匹配从左往右匹配，要使用order_no走索引，必须查询条件携带shop_id或者索引(shop_id,order_no)调换前后顺序。

**2、隐式转换**

**1）索引**

```sql
KEY `idx_mobile` (`mobile`)
```

**2）SQL语句**

```sql
select * from _user where mobile=12345678901
```

隐式转换相当于在索引上做运算，会让索引失效。mobile是字符类型，使用了数字，应该使用字符串匹配，否则MySQL会用到隐式替换，导致索引失效。

**3、大分页**

**1）索引**

```sql
KEY `idx_a_b_c` (`a`, `b`, `c`)
```

**2）SQL语句**

```sqlite
select * from _t where a = 1 and b = 2 order by c desc limit 10000, 10;
```

对于大分页的场景，可以优先让产品优化需求，如果没有优化的，有如下两种优化方式：

- 一种是把上一次的最后一条数据，也即上面的c传过来，然后做“c < xxx”处理，但是这种一般需要改接口协议，并不一定可行；‘
- 另一种是采用延迟关联的方式进行处理，减少SQL回表，但是要记得索引需要完全覆盖才有效果，SQL改动如下：

```sql
select t1.* from _t t1, (select id from _t where a = 1 and b = 2 order by c desc limit 10000, 10) t2 where t1.id = t2.id;
```

**4、in + order by**

**1）索引**

```sql
KEY `idx_shopid_status_created` (`shop_id`, `order_status`, `created_at`)
```

**2）SQL语句**

```sql
select * from _order where shop_id = 1 and order_status in (1, 2, 3) order by created_at desc limit 10
```

in查询在MySQL底层是通过n*m的方式去搜索，类似union，但是效率比union高。

in查询在进行cost代价计算时（代价 = 元组数 * IO平均值），是通过将in包含的数值，一条条去查询获取元组数的，因此这个计算过程会比较的慢，所以MySQL设置了个临界值(eq_range_index_dive_limit)，5.6之后超过这个临界值后该列的cost就不参与计算了。因此会导致执行计划选择不准确。默认是200，即in条件超过了200个数据，会导致in的代价计算存在问题，可能会导致Mysql选择的索引不准确。

**3）处理方式**

可以(order_status, created_at)互换前后顺序，并且调整SQL为延迟关联。

**5、范围查询阻断，后续字段不能走索引**

**1）索引**

```sql
KEY `idx_shopid_created_status` (`shop_id`, `created_at`, `order_status`)
```

**2）SQL语句**

```sql
select * from _order where shop_id = 1 and created_at > '2021-01-01 00:00:00' and order_status = 10
```

范围查询还有“IN、between”。

**6、不等于、不包含不能用到索引的快速搜索**

可以用到ICP

```sql
select * from _order where shop_id=1 and order_status not in (1,2)
select * from _order where shop_id=1 and order_status != 1
```

在索引上，避免使用NOT、!=、<>、!<、!>、NOT EXISTS、NOT IN、NOT LIKE等。

**7、优化器选择不使用索引的情况**

如果要求访问的数据量很小，则优化器还是会选择辅助索引，但是当访问的数据占整个表中数据的蛮大一部分时（一般是20%左右），优化器会选择通过聚集索引来查找数据。

```sql
select * from _order where  order_status = 1
```

查询出所有未支付的订单，一般这种订单是很少的，即使建了索引，也没法使用索引。

**8、复杂查询**

```sql
select sum(amt) from _t where a = 1 and b in (1, 2, 3) and c > '2020-01-01';
select * from _t where a = 1 and b in (1, 2, 3) and c > '2020-01-01' limit 10;
```

如果是统计某些数据，可能改用数仓进行解决；

如果是业务上就有那么复杂的查询，可能就不建议继续走SQL了，而是采用其他的方式进行解决，比如使用ES等进行解决。

**9、asc和desc混用**

```sql
select * from _t where a=1 order by b desc, c asc
```

desc 和asc混用时会导致索引失效。

**10、大数据**

对于推送业务的数据存储，可能数据量会很大，如果在方案的选择上，最终选择存储在MySQL上，并且做7天等有效期的保存。

那么需要注意，频繁的清理数据，会照成数据碎片，需要联系DBA进行数据碎片处理。

### 日活百万以内，统计用户新增活跃留存方案

#### 背景

产品上线后，出于运营的需要，我们要对用户进行跟踪，分析用户数据。本文要介绍的是如何统计用户新增数、活跃数和留存率，时间跨度是天，即统计每日新增（DNU），日活（DAU）和某日新增的一批用户在接下来的一段时间内每天活跃的百分比。

#### 使用范围

本方案适用于用户量不太大（日活在百万以内，日活百万以上不是不能用，只是在统计数据时耗时太长不太合适），尤其适合小团队或个人开发者（比如你公司服务端接口开发是你，运维也是你，现在老板又来叫你做数据报表）。如果你的产品的日活有几百万甚至几千万或过亿，这样的产品当然是完全可以养一个大数据部门的，本方案并不适用这种情况。

#### 涉及到的工具和技术点

##### shell脚本

本方案需要你懂一点儿shell，起码能看懂，也要求你知道怎么写crontab定时任务。

##### MySQL

本方案需要你熟练使用sql，知道怎么定义存储过程，知道分区表的概念和用法。

#### 实现过程

##### 一、目标

由于数据量是不断增加的，所以我们的目标是要把原始数据聚合成一张可以直接用一条select语句就可以查看每日新增、日活和留存率的表，并且只能做单表查询，否则当数据量增大时，联表查询的速度会大大下降。而且为了防止出错，我们的数据还需要可以重跑但是不会影响到已存在的数据。

最终呈现给运营人员看到的数据是这样的：

新增-活跃表

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XADPuASuGoqpkmTwfqEzz86Ba6a3ZOAKZibp5OhjPyCA5NM22GnxVFpUMsJm3lFwHbPJovEkEJaJWQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

用户留存表

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XADPuASuGoqpkmTwfqEzz86l5HoMJuskSRLMmZACAvqqlteITacFA9Jhxq0Gr2IniaBibLjzcXQnynQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

简单解释一下上面两个表的结构：因为我们是按天统计的，所以日期都是以天为单位，用户可能有不同的国家或地区，不同版本，不同手机型号等等，所以就有了各个维度。用户留存表的数据要注意，比如今天是2022年2月9日，那么就只能统计到9号的新增，9号新增用户的日次留存是10号才能统计到的，但是8号新增用户的次日留存在今天（也就是9号）就统计出来了，所以留存的数据是一个阶梯形状的。

##### 二、收集数据

为了方便介绍本方案，这里假设只有日期、国家、版本号三个维度。

收集数据的下一步是数据入库，为了方便，需要把数据格式进行转换。因为服务端接口现在一般都是使用json格式的数据进行通信，如果直接把json格式的数据输出到日志文件，处理起来会非常麻烦，所以需要在服务端接收到统计日志时，把数据输出到单独的日志文件中，还要按照MySQL的load命令可以识别的数据格式。

在输出日志之前，先确定好都需要哪些数据，这里需要的数据如下：

- `ts`：timestamp，时间戳。服务端接收到日志的时间，格式是yyyy-MM-DD HH:mm:ss。
- `device_id`：设备id，这里是用来唯一标识用户的一个字符串，比如在android设备上可以用android id，总之这个字段是用来确定一台设备的，要保证不同的设备设备id不同。
- `country`：用户所在的国家。如果你是只做一个国家的，比如只做国内市场，也可以把这个字段换成省份或者城市，总之根据运营需求去改变。
- `version`：应用版本号，一般是一个整数。

于是就可以确定日志的格式如下：

```
2022-02-09 13:14:15||aaaaaa||CN||100
2022-02-09 13:14:16||bbbbbb||US||100
2022-02-09 13:14:17||cccccc||NL||100
```

也就是一条数据占一行，字段之间使用双竖线分隔，当然这里不一定是双竖线分隔，也可以换成其它的，原则是字符数少而且不能被字段的值包含，不然在数据入库时会出现字段不对应的问题。

再考虑两个方面：

- 如果数据量较大要怎么处理？
- 可能有的字段的长度没法一下子确定怎么处理？
- 保留数据的策略应该怎样设置？

第1个问题，当数据量大时，可以考虑把日志文件切割成更小的时间段，比如每小时一个日志文件，然后下一小时就把上一个小时的数据入库。

第2个问题，原始数据表的字段长度定义得大一些，做到即使以后字段有变化，也可以适应。

第3个问题，因为我们的目标是跑出最后的报表，所以不可能一直保存着所有的原始日志数据，为了防止出错，可能只是保留最近几天的，一个简单的策略是在每次日志数据入库前用delete语句把前几天的数据删除了，但是直接使用delete有两个问题：一是MySQL要扫描全表删除数据，比较耗时；二是MySQL的delete + where删除可能只是假删除，磁盘不会立即释放。

所以这里使用分区表来实现，每天的数据作为一个分区，删除数据时直接删除分区，数据入库时先创建分区。

于是得到原始数据表的DDL如下：

```sql
CREATE TABLE `st_base` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'ID',
  `dd` int(11) NOT NULL DEFAULT '0' COMMENT '天数，格式是yyyyMMddHH',
  `ts` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '' COMMENT '时间戳',
 `device_id` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '' COMMENT '设备id',
 `country` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '' COMMENT '国家',
 `version` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '' COMMENT '版本号',
 PRIMARY KEY (`id`,`dd`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='原始数据表'
/*!50100 PARTITION BY LIST (dd)
(PARTITION p20220209 VALUES IN (20220209) ENGINE = InnoDB) */
```

##### 三、数据入库

有了格式化的日志文件和数据表，就可以通过shell脚本把数据入库了。步骤如下：

- 删除历史日志的分区
- 删除执行日期的分区（这一步在重跑数据很有用）
- 创建执行日期的分区
- 使用MySQL的load命令把数据从日志文件加载到数据库中

这里只说一下重要的命令：

1,删除和创建分区可以分别使用下面两个命令：

```
drop_sql="alter table st_base drop partition pxxxxxxxx" # 这里的xxxxxxxx要根据执行日期转换一下
add_sql="alter table st_base add partition (partition pxxxxxxxx values in (xxxxxxxx) engine=innodb)"

mysql -u${username} -p${password} -D${database} -e "${drop_sql}"

mysql -u${username} -p${password} -D${database} -e "${add_sql}"
```

上面使用mysql命令指定了用户名、密码、数据库名和sql语句（-e参数）

2,从文件加载数据入库

```
log_file=xxxx #日志文件名
dd=xxxxxxxx #执行日期
load_sql="load data local infile '${log_file}' ignore into table st_base fields terminated by '||' lines terminated by '\n' (ts,device_id,country,version) set dd='${dd}'"
mysql -u${username} -p${password} -D${database} -e "${load_sql}"
```

3,定时任务

因为我们是每天入库一次，所以可以在每天的0时10分去跑上面的脚本任务。假设上面的脚本文件保存为st_base.sh

可以通过crontab -e编辑定时任务：

```
10 0 * * * /path/to/job/st_base.sh
```

当然最好的做法是把执行日期当做脚本的参数传入，这样可以实现重跑某天的数据了。

##### 四、清洗数据

在上一步得到了原始数据之后，接下来的工作都可以在MySQL中完成，首先要清洗数据。

这一步的目的有两个：

- 确定好数据类型
- 数据去重

先创建一个临时表tmp_base，这个表用来转换数据类型，如果有一些字段的值需要转换的也可以在这里做（举个例子：假如客户端获取到的国家有几种途径，分别是获取了sim卡国家，网络国家，手机国家，到了服务端后服务器根据客户端的ip也解析出了一个国家，但是运营的时候可能只需要一个最接近用户的真实国家，那么就可以按照优先级来确定，当然本文没有多个国家的问题），DDL如下：

```sql
CREATE TABLE `tmp_base` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT 'ID',
  `dt` date NOT NULL COMMENT '日期',
 `device_id` varchar(32) COLLATE utf8_bin NOT NULL DEFAULT '' COMMENT '设备id',
 `country` varchar(8) COLLATE utf8_bin DEFAULT NULL,
  `version` int(11) NOT NULL DEFAULT '0' COMMENT '版本号',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='用户基础临时表'
```

再创建一个用户总表total_base，这个表用来存放所有用户的数据，每个用户只有一条数据，DDL如下：

```sql
CREATE TABLE `total_base` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'ID',
  `dt` date NOT NULL COMMENT '新增日期',
  `device_id` varchar(32) COLLATE utf8_bin NOT NULL DEFAULT '' COMMENT '设备id',
 `country` varchar(8) COLLATE utf8_bin DEFAULT NULL,
  `version` int(11) NOT NULL DEFAULT '0' COMMENT '版本号',
  PRIMARY KEY (`id`),
  UNIQUE KEY `device` (`device_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='用户总表';
```

创建一个流水表flow_base，同样以日期作为分区字段，DDL如下：

```sql
CREATE TABLE `flow_base` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'ID',
  `dt` date NOT NULL DEFAULT '2022-01-01' COMMENT '日期',
 `device_id` varchar(32) COLLATE utf8_bin NOT NULL DEFAULT '' COMMENT '设备id',
 `country` varchar(8) COLLATE utf8_bin DEFAULT NULL,
  `version` int(11) NOT NULL DEFAULT '0' COMMENT '版本号',
  `rdt` date NOT NULL DEFAULT '2022-01-01' COMMENT '用户注册日期',
  `dd` int(11) NOT NULL DEFAULT '0' COMMENT '日期(yyyyMMdd)，用来做分区',
  PRIMARY KEY (`id`,`dd`),
  UNIQUE KEY `unique` (`dt`,`device_id`,`dd`)
) ENGINE=InnoDB CHARSET=utf8 COLLATE=utf8_bin COMMENT='用户基础流水表'
/*!50100 PARTITION BY LIST (dd)
(PARTITION p20220209 VALUES IN (20220209) ENGINE = InnoDB) */
```

注意到流水表flow_base中有一个rdt的字段，这字段是用来存放这个用户的注册日期，方便后面统计留存使用的。

准备好表结构之后，开始清洗数据。清洗数据使用MySQL的存储过程功能，创建一个存储过程sp_data_cleaning，在这个存储过程中，需要做以下几件事：

- 把原始数据表st_base中的数据清洗到临时表tmp_base，如果有字段的值需要转换也在这一步做。
- 把临时表tmp_base中的用户添加到用户总表total_base中。
- 把临时表tmp_base中的数据添加到流水表中，并且联合用户总表，把用户的注册日期也填充好。

于是可以得到存储过程sp_data_cleaning的DDL如下：

```sql
CREATE PROCEDURE `sp_data_cleaning`(IN v_dt VARCHAR(10))
BEGIN
 # 变量
 declare pname varchar(10);
 declare v_is_pname_exists int;
 
 # 清除tmp_base表数据
    truncate table tmp_base;

 # 清洗数据
  insert into tmp_base(
   `dt`,
    `device_id`,
   `country`,
    `version`
   )
  select
   v_dt,
   `device_id`,
    `country`,
    `version`
   from `st_base`
   where `dd` = replace(v_dt,'-','');
   
 # 数据加入用户总表
  insert ignore into total_base(
   `dt`,
   `device_id`,
   `country`,
   `version`
   )
  select
   `dt`,
   `device_id`,
   `country`,
   `version`
  from tmp_base;
  
 # 给流水表创建分区
 select concat('p', replace(v_dt, '-', '')) into pname;
 
 # 查找是否已经存在执行日期的分区
 select max(a) into v_is_pname_exists
 from (
  select 1 as a from information_schema.PARTITIONS 
  where `TABLE_SCHEMA` = 'your_database_name' 
   and `TABLE_NAME` = 'flow_base'
   and `PARTITION_NAME`=pname
  union all
  select 0 
  ) t;
 
 # 如果已经存在先删除
 if v_is_pname_exists=1 then
 set @drop_sql=concat('alter table flow_base drop partition ', pname);
 prepare stmt from @drop_sql;
 execute stmt;
 deallocate prepare stmt;
 end if;

 # 创建分区
 set @add_sql=concat('alter table flow_base add partition (partition ', pname, ' values in (', v_date, ') ENGINE = InnoDB)');
 prepare stmt from @add_sql;
 execute stmt;
 deallocate prepare stmt;  

 # 数据加入流水表
  insert ignore into flow_base(
   `dt`,
   `device_id`,
   `country`,
   `version`,
   `rdt`,
   `dd`
   )
  select
   v_dt,
   t1.`device_id`,
   t1.`country`,
   t1.`version`,
   t2.`dt`,
   replace(v_dt, '-', '')
  from tmp_base t1
  left outer join total_base t2
  on (t1.`device_id`=t2.`device_id`);
END
```

##### 五、数据聚合

经过上面几个步骤的处理，现在已经得到了半成品的数据，可以进行聚合了。根据第一步的目标报表，可以确定两个表的结构：一个是用户的新增-活跃表，另一个是用户的留存表，DDL如下：

新增-活跃表：

```sql
CREATE TABLE `rpt_base_active` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'ID',
  `dt` date NOT NULL DEFAULT '2022-01-01' COMMENT '日期',
 `country` varchar(8) COLLATE utf8_bin DEFAULT NULL,
  `version` int(11) NOT NULL DEFAULT '0' COMMENT '版本号',
  `new_users` smallint(4) NOT NULL DEFAULT '0' COMMENT '新增数',
  `active_users` smallint(4) NOT NULL DEFAULT '0' COMMENT '活跃数',
  PRIMARY KEY (`id`),
  KEY `index1` (`dt`),
  KEY `index3` (`country`,`version`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='用户新增活跃表'
```

用户留存表（这里假设只看7天的留存情况，如果需要看更多留存天数，可以自行修改）：

```sql
CREATE TABLE `rpt_base` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'ID',
  `dt` date NOT NULL DEFAULT '2022-01-01' COMMENT '日期',
 `country` varchar(8) COLLATE utf8_bin DEFAULT NULL,
  `version` int(11) NOT NULL DEFAULT '0' COMMENT '版本号',
  `d0` smallint(4) NOT NULL DEFAULT '0' COMMENT '新增数',
  `d1` smallint(4) NOT NULL DEFAULT '0' COMMENT '次日留存数',
  `d2` smallint(4) NOT NULL DEFAULT '0' COMMENT '2日留存数',
  `d3` smallint(4) NOT NULL DEFAULT '0' COMMENT '3日留存数',
  `d4` smallint(4) NOT NULL DEFAULT '0' COMMENT '4日留存数',
  `d5` smallint(4) NOT NULL DEFAULT '0' COMMENT '5日留存数',
  `d6` smallint(4) NOT NULL DEFAULT '0' COMMENT '6日留存数',
  `d7` smallint(4) NOT NULL DEFAULT '0' COMMENT '7日留存数',
  PRIMARY KEY (`id`),
  KEY `index1` (`dt`),
  KEY `index3` (`country`,`version`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='用户留存表'
```

> 注意，以上两个表的索引创建并不是固定的，需要根据运营的实际情况去创建相关的索引。

在跑数据之前，先聚合一下执行日期的数据，创建一个临时表a_flow_base，这个表用来初步聚合数据，DDL如下：

```sql
CREATE TABLE `a_flow_base` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'ID',
  `dt` date NOT NULL DEFAULT '2022-01-01' COMMENT '日期',
 `country` varchar(8) COLLATE utf8_bin DEFAULT NULL,
  `version` int(11) NOT NULL DEFAULT '0' COMMENT '应用版本号',
  `rdt` date NOT NULL DEFAULT '2022-01-01' COMMENT '用户注册日期',
  `rdays` smallint(4) NOT NULL DEFAULT '0' COMMENT '留存天数',
  `users` smallint(4) NOT NULL DEFAULT '0' COMMENT '用户数',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='用户基础数据聚合表'
```

首先初步聚合用户数据，创建一个存储过程sp_a_flow_base，DDL如下：

```sql
CREATE PROCEDURE `sp_a_flow_base`(in v_dt char(10))
BEGIN
 declare d0 date;
 declare d1 date;
 declare d2 date;
 declare d3 date;
 declare d4 date;
 declare d5 date;
 declare d6 date;
 declare d7 date;
 
 select date_sub(v_dt, interval 0 day) into d0;
 select date_sub(v_dt, interval 1 day) into d1;
 select date_sub(v_dt, interval 2 day) into d2;
 select date_sub(v_dt, interval 3 day) into d3;
 select date_sub(v_dt, interval 4 day) into d4;
 select date_sub(v_dt, interval 5 day) into d5;
 select date_sub(v_dt, interval 6 day) into d6;
 select date_sub(v_dt, interval 7 day) into d7;
 
 # 清除a_flow_base表数据
 truncate table a_flow_base;
 
 insert into a_flow_base(
  `dt`,
  `country`,
   `version_code`,
   `rdt`,
   `rdays`,
   `users`
  )
 select 
  t1.`dt`,
   t1.`country`,
   t1.`version`,
   t1.`rdt`,
   datediff(t1.`dt`, t1.`rdt`) as rdays,
   count(*) as users
 from flow_base t1
 where t1.`dt` in (d0,d1,d2,d3,d4,d5,d6,d7)
 group by 
  t1.`dt`,
   t1.`country`,
   t1.`version`,
   t1.`rdt`;
END
```

初步聚合了数据后，开始正式聚合数据，创建一个存储过程sp_rpt_base，DDL如下：

```sql
CREATE PROCEDURE `sp_rpt_base`(in v_dt char(10))
BEGIN
 declare d0 date;
 declare d1 date;
 declare d2 date;
 declare d3 date;
 declare d4 date;
 declare d5 date;
 declare d6 date;
 declare d7 date;
 
 select date_sub(v_dt, interval 0 day) into d0;
 select date_sub(v_dt, interval 1 day) into d1;
 select date_sub(v_dt, interval 2 day) into d2;
 select date_sub(v_dt, interval 3 day) into d3;
 select date_sub(v_dt, interval 4 day) into d4;
 select date_sub(v_dt, interval 5 day) into d5;
 select date_sub(v_dt, interval 6 day) into d6;
 select date_sub(v_dt, interval 7 day) into d7;
 
 # 删除数据
 delete from rpt_base_active where `dt` = v_dt;
 
 insert into rpt_base_active (
  `dt`,
  `country`,
   `version`,
   `new_users`,
   `active_users`
  )
 select
  `dt`,
  `country`,
  `version`,
  sum(if(`dt`=`rdt`, 1, 0)) as `new_users`,
  sum(1) as `active_users`
 from flow_base
 where dt=v_dt
 group by 
  `dt`,
  `country`,
  `version`
  ;

 # 删除数据
 delete from rpt_base where `dt` in (d0,d1,d2,d3,d4,d5,d6,d7);
 
 insert into rpt_base(
  `dt`,
  `country`,
   `version`,
   `d0`,
   `d1`,
   `d2`,
   `d3`,
   `d4`,
   `d5`,
   `d6`,
   `d7`
  )
 select
  t1.`rdt`,
  t1.`country`,
  t1.`version`,
  sum(case when t1.`rdays`=0 then t1.`users` else 0 end) as d0,
  sum(case when t1.`rdays`=1 then t1.`users` else 0 end) as d1,
  sum(case when t1.`rdays`=2 then t1.`users` else 0 end) as d2,
  sum(case when t1.`rdays`=3 then t1.`users` else 0 end) as d3,
  sum(case when t1.`rdays`=4 then t1.`users` else 0 end) as d4,
  sum(case when t1.`rdays`=5 then t1.`users` else 0 end) as d5,
  sum(case when t1.`rdays`=6 then t1.`users` else 0 end) as d6,
  sum(case when t1.`rdays`=7 then t1.`users` else 0 end) as d7
  from a_flow_base t1
  group by
   t1.`rdt`,
   t1.`country`,
    t1.`version`
   ;
   
END
```

为了方便调用整个过程，可以再创建一个存储过程，把全过程写在一起，一次执行。创建一个存储过程sp_user，DDL如下：

```sql
CREATE PROCEDURE `sp_user`(in v_dt char(10))
BEGIN

 call sp_tmp_base(v_dt);
 call sp_data_cleaning(v_dt);
 call sp_a_flow_base(v_dt);
 call sp_rpt_base(v_dt);
 
END
```

这样，就可以添加定时任务每天定时跑前一天的数据了。

#### 写在最后

流水表flow_base应该保留几天的数据？

这个看你的用户留存表需要看多少天留存数据，如果你要看7日留存，那么保留最近8天的数据，如果是想看30天留存，就保留最近31天的数据，依次类推。

如果运营人员或老板9点半上班，每天凌晨的0点开始跑前一天的数据，你将有9个半小时来跑前一天的数据。当然如果一天的数据要跑2个小时以上，还是考虑用Hadoop来做吧。

### SQL 的10个高级概念(With/CASE/窗口函数)

随着数据量持续增长，对合格数据专业人员的需求也会增长。具体而言，对SQL流利的专业人士的需求日益增长，而不仅仅是在初级层面。

因此，Stratascratch的创始人Nathan Rosidi以及我觉得我认为10个最重要和相关的中级到高级SQL概念。

#### 1.常见表表达式（CTEs）

如果您想要查询子查询，那就是CTEs施展身手的时候 - CTEs基本上创建了一个临时表。

使用常用表表达式（CTEs）是模块化和分解代码的好方法，与您将文章分解为几个段落的方式相同。

请在Where子句中使用子查询进行以下查询。

```sql
SELECT   
 name,  
 salary   
FROM  
 People   
WHERE  
 NAME IN ( SELECT DISTINCT NAME FROM population WHERE country = "Canada" AND city = "Toronto" )   
 AND salary >= (  
 SELECT  
  AVG( salary )   
 FROM  
  salaries   
WHERE  
 gender = "Female")  
```

这似乎似乎难以理解，但如果在查询中有许多子查询，那么怎么样？这就是CTEs发挥作用的地方。

```sql
with toronto_ppl as (  
   SELECT DISTINCT name  
   FROM population  
   WHERE country = "Canada"  
         AND city = "Toronto"  
)  
, avg_female_salary as (  
   SELECT AVG(salary) as avgSalary  
   FROM salaries  
   WHERE gender = "Female"  
)  
SELECT name  
       , salary  
FROM People  
WHERE name in (SELECT DISTINCT FROM toronto_ppl)  
      AND salary >= (SELECT avgSalary FROM avg_female_salary)  
```

现在很清楚，Where子句是在多伦多的名称中过滤。如果您注意到，CTE很有用，因为您可以将代码分解为较小的块，但它们也很有用，因为它允许您为每个CTE分配变量名称（即toronto_ppl和avg_female_salary）

同样，CTEs允许您完成更高级的技术，如创建递归表。

#### 2.递归CTEs.

递归CTE是引用自己的CTE，就像Python中的递归函数一样。递归CTE尤其有用，它涉及查询组织结构图，文件系统，网页之间的链接图等的分层数据，尤其有用。

递归CTE有3个部分：

- 锚构件：返回CTE的基本结果的初始查询
- 递归成员：引用CTE的递归查询。这是所有与锚构件的联盟
- 停止递归构件的终止条件

以下是获取每个员工ID的管理器ID的递归CTE的示例：

```sql
with org_structure as (  
   SELECT id  
          , manager_id  
   FROM staff_members  
   WHERE manager_id IS NULL  
   UNION ALL  
   SELECT sm.id  
          , sm.manager_id  
   FROM staff_members sm  
   INNER JOIN org_structure os  
      ON os.id = sm.manager_id  
```

#### 3.临时函数

如果您想了解有关临时函数的更多信息，请检查此项，但知道如何编写临时功能是重要的原因：

- 它允许您将代码的块分解为较小的代码块
- 它适用于写入清洁代码
- 它可以防止重复，并允许您重用类似于使用Python中的函数的代码。

考虑以下示例：

```sql
SELECT name  
       , CASE WHEN tenure < 1 THEN "analyst"  
              WHEN tenure BETWEEN 1 and 3 THEN "associate"  
              WHEN tenure BETWEEN 3 and 5 THEN "senior"  
              WHEN tenure > 5 THEN "vp"  
              ELSE "n/a"  
         END AS seniority   
FROM employees  
```

相反，您可以利用临时函数来捕获案例子句。

```sql
CREATE TEMPORARY FUNCTION get_seniority(tenure INT64) AS (  
   CASE WHEN tenure < 1 THEN "analyst"  
        WHEN tenure BETWEEN 1 and 3 THEN "associate"  
        WHEN tenure BETWEEN 3 and 5 THEN "senior"  
        WHEN tenure > 5 THEN "vp"  
        ELSE "n/a"  
   END  
);  
SELECT name  
       , get_seniority(tenure) as seniority  
FROM employees  
```

通过临时函数，查询本身更简单，更可读，您可以重复使用资历函数！

#### 4.使用CASE WHEN枢转数据

您很可能会看到许多要求在陈述时使用CASE WHEN的问题，这只是因为它是一种多功能的概念。如果要根据其他变量分配某个值或类，则允许您编写复杂的条件语句。

较少众所周知，它还允许您枢转数据。例如，如果您有一个月列，并且您希望为每个月创建一个单个列，则可以使用语句追溯数据的情况。

示例问题：编写SQL查询以重新格式化表，以便每个月有一个收入列。

```sql
Initial table:    
+------+---------+-------+    
| id   | revenue | month |    
+------+---------+-------+    
| 1    | 8000    | Jan   |    
| 2    | 9000    | Jan   |    
| 3    | 10000   | Feb   |    
| 1    | 7000    | Feb   |    
| 1    | 6000    | Mar   |    
+------+---------+-------+    
  
Result table:    
+------+-------------+-------------+-------------+-----+-----------+    
| id   | Jan_Revenue | Feb_Revenue | Mar_Revenue | ... | Dec_Revenue |    
+------+-------------+-------------+-------------+-----+-----------+    
| 1    | 8000        | 7000        | 6000        | ... | null        |    
| 2    | 9000        | null        | null        | ... | null        |    
| 3    | null        | 10000       | null        | ... | null        |    
+------+-------------+-------------+-------------+-----+-----------+  
```

#### 5.EXCEPT vs NOT IN

除了几乎不相同的操作。它们都用来比较两个查询/表之间的行。所说，这两个人之间存在微妙的细微差别。

首先，除了过滤删除重复并返回不同的行与不在中的不同行。

同样，除了在查询/表中相同数量的列，其中不再与每个查询/表比较单个列。

#### 6.自联结

一个SQL表自行连接自己。你可能会认为没有用，但你会感到惊讶的是这是多么常见。在许多现实生活中，数据存储在一个大型表中而不是许多较小的表中。在这种情况下，可能需要自我连接来解决独特的问题。

让我们来看看一个例子。

示例问题：给定下面的员工表，写出一个SQL查询，了解员工的工资，这些员工比其管理人员工资更多。对于上表来说，Joe是唯一一个比他的经理工资更多的员工。

```sql
+----+-------+--------+-----------+    
| Id | Name  | Salary | ManagerId |    
+----+-------+--------+-----------+    
| 1  | Joe   | 70000  | 3         |    
| 2  | Henry | 80000  | 4         |    
| 3  | Sam   | 60000  | NULL      |    
| 4  | Max   | 90000  | NULL      |    
+----+-------+--------+-----------+Answer:    
SELECT    
    a.Name as Employee    
FROM    
    Employee as a    
    JOIN Employee as b on a.ManagerID = b.Id    
WHERE a.Salary > b.Salary  
```

#### 7.Rank vs DenseRank vs RowNumber

它是一个非常常见的应用，对行和价值进行排名。以下是公司经常使用排名的一些例子：

- 按购物，利润等数量排名最高值的客户
- 排名销售数量的顶级产品
- 以最大的销售排名顶级国家
- 排名在观看的分钟数，不同观众的数量等观看的顶级视频。

在SQL中，您可以使用几种方式将“等级”分配给行，我们将使用示例进行探索。考虑以下Query和结果：

```sql
SELECT Name    
 , GPA    
 , ROW_NUMBER() OVER (ORDER BY GPA desc)    
 , RANK() OVER (ORDER BY GPA desc)    
 , DENSE_RANK() OVER (ORDER BY GPA desc)    
FROM student_grades  
```

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/RQueXibgo0KPq4lvUrW3icFhUiaQNEiaaM5zOCdiaBuJnR5CGuY7eclqRNibUhcibicsic0Bs2ar9bqEwBfMVOtNZkjlVfQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)图片

ROW_NUMBER()返回每行开始的唯一编号。当存在关系时（例如，BOB vs Carrie），ROW_NUMBER()如果未定义第二条标准，则任意分配数字。

Rank()返回从1开始的每行的唯一编号，除了有关系时，等级()将分配相同的数字。同样，差距将遵循重复的等级。

dense_rank()类似于等级()，除了重复等级后没有间隙。请注意，使用dense_rank()，Daniel排名第3，而不是第4位()。

#### 8.计算Delta值

另一个常见应用程序是将不同时期的值进行比较。例如，本月和上个月的销售之间的三角洲是什么？或者本月和本月去年这个月是什么？

在将不同时段的值进行比较以计算Deltas时，这是Lead()和LAG()发挥作用时。

这是一些例子：

```sql
# Comparing each month's sales to last month    
SELECT month    
       , sales    
       , sales - LAG(sales, 1) OVER (ORDER BY month)    
FROM monthly_sales    
# Comparing each month's sales to the same month last year    
SELECT month    
       , sales    
       , sales - LAG(sales, 12) OVER (ORDER BY month)    
FROM monthly_sales  
```

#### 9.计算运行总数

如果你知道关于row_number()和lag()/ lead()，这可能对您来说可能不会惊喜。但如果你没有，这可能是最有用的窗口功能之一，特别是当您想要可视化增长！

使用具有SUM（）的窗口函数，我们可以计算运行总数。请参阅下面的示例：

```sql
SELECT Month    
       , Revenue    
       , SUM(Revenue) OVER (ORDER BY Month) AS Cumulative    
FROM monthly_revenue  
```

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/RQueXibgo0KPq4lvUrW3icFhUiaQNEiaaM5z4DHNMibzClf3GnXeEC9ibbPIkSOEk4akVXgl5o9JejcNcheFfJwtSBaQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)图片

#### 10.日期时间操纵

您应该肯定会期望某种涉及日期时间数据的SQL问题。例如，您可能需要将数据分组组或将可变格式从DD-MM-Yyyy转换为简单的月份。

您应该知道的一些功能是：

- 提炼
- 日元
- date_add，date_sub.
- date_trunc.

示例问题：给定天气表，写一个SQL查询，以查找与其上一个（昨天）日期相比的温度较高的所有日期的ID。

```sql
+---------+------------------+------------------+    
| Id(INT) | RecordDate(DATE) | Temperature(INT) |    
+---------+------------------+------------------+    
|       1 |       2015-01-01 |               10 |    
|       2 |       2015-01-02 |               25 |    
|       3 |       2015-01-03 |               20 |    
|       4 |       2015-01-04 |               30 |    
+---------+------------------+------------------+Answer:    
SELECT    
    a.Id    
FROM    
    Weather a,    
    Weather b    
WHERE    
    a.Temperature > b.Temperature    
    AND DATEDIFF(a.RecordDate, b.RecordDate) = 1  
```

就这样！我希望这有助于您在面试准备中 - 我相信，如果您知道这10个内部概念，那么在那里大多数SQL问题时，你会做得很好。

(本文由闻数起舞翻译自Dimitris Poulopoulos的文章《Ten Advanced SQL Concepts You Should Know for Data Science Interviews》，转载请注明出处，原文链接：
https://towardsdatascience.com/ten-advanced-sql-concepts-you-should-know-for-data-science-interviews-4d7015ec74b0)

### SQL中的 CASE 表达式妙用这么多，你知道吗？

#### CASE表达式 之概念

相信大家都用过CASE表达式，尤其是做一些统计功能的时候，用的特别多，可真要说什么是 CASE表达式，我估计还真没几个人能清楚的表述出来。

CASE表达式和 “2+1” 或者 “120/3” 这样的表达式一样，是一种进行运算的功能，正如CASE（情况）这个词的含义一样，用于区分情况，在有条件分歧的时候使用它。

CASE表达式是从 SQL-92 标准开始被引入的，可能因为它是相对较新的技术，所以尽管使用起来非常便利，但其真正的价值却并不怎么为人所知。很多人不用它，或者用它的简略版函数，例如 DECODE（Oracle）、IF（MySQL）等。然而，CASE表达式也许是 SQL-92 标准里加入的最有用的特性，如果能用好它，那么 SQL 能解决的问题就会更广泛，写法也会更加漂亮，而且，因为 CASE表达式 是不依赖于具体数据库的技术，所以可以提高 SQL 代码的可移植性。

**基本格式如下**

```sql
-- 简单 CASE表达式  
CASE 列(或表达式)  
     WHEN <匹配值1> THEN <表达式>  
     WHEN <匹配值2> THEN <表达式>  
     ......  
     ELSE <表达式>  
END  
  
-- 搜索 CASE表达式  
CASE WHEN <判断表达式> THEN <表达式>  
     WHEN <判断表达式> THEN <表达式>  
     WHEN <判断表达式> THEN <表达式>  
     ......  
     ELSE <表达式>  
END  
  
  
-- 简单 CASE表达式 示例  
CASE sex  
    WHEN '1' THEN '男'  
    WHEN '2' THEN '女'  
    ELSE '其他'   
END  
  
-- 搜索CASE表达式 示例  
CASE WHEN sex = '1' THEN '男'  
     WHEN sex = '2' THEN '女'  
     ELSE '其他'   
END  
```

CASE表达式 的 ELSE子句 可以省略，但推荐不要省略，省略了可能会出现我们意料之外的结果。END不能省，必须有。当 WHEN子句 为真时，CASE表达式 的真假值判断就会中止，而剩余的 WHEN子句会被忽略。为了避免引起不必要的混乱，使用 WHEN子句 时要注意条件的排他性。

简单CASE表达式正如其名，写法简单，但能实现的功能比较有限。简单CASE表达式能写的条件，搜索CASE表达式也能写，所以基本上采用搜索CASE表达式的写法。

#### CASE表达式 之妙用

上面讲了 CASE表达式 的理论知识，感觉不痛不痒，那么接下来我们进入实战篇，结合一些场景来看看 CASE表达式 的妙用

##### 行转列

可能我们用的更多的是 IF（MySQL）或 DECODE（Oracle），但这两者都不是标准的 SQL，更推荐大家用 CASE表达式，移植性更高

假设我们有如下表，以及如下数据

```sql
CREATE TABLE t_customer_credit (  
    id INT(11) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '自增主键',  
    login_name VARCHAR(50) NOT NULL COMMENT '登录名',  
    credit_type TINYINT(1) NOT NULL COMMENT '额度类型，1:自由资金,2:冻结资金,3:优惠',  
    amount DECIMAL(22,6) NOT NULL DEFAULT '0.00000' COMMENT '额度值',  
    create_by VARCHAR(50) NOT NULL COMMENT '创建者',  
    create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',  
    update_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '创建时间',  
    update_by VARCHAR(50) NOT NULL COMMENT '修改者',  
  PRIMARY KEY (id)  
);  
INSERT INTO `t_customer_credit` VALUES (1, 'zhangsan', 1, 550.000000, 'system', '2019-7-7 11:30:09', '2019-7-8 20:21:05', 'system');  
INSERT INTO `t_customer_credit` VALUES (2, 'zhangsan', 2, 0.000000, 'system', '2019-7-7 11:30:09', '2019-7-7 11:30:09', 'system');  
INSERT INTO `t_customer_credit` VALUES (3, 'zhangsan', 3, 0.000000, 'system', '2019-7-7 11:30:09', '2019-7-7 11:30:09', 'system');  
INSERT INTO `t_customer_credit` VALUES (4, 'lisi', 1, 0.000000, 'system', '2019-7-7 11:30:09', '2019-7-7 11:30:09', 'system');  
INSERT INTO `t_customer_credit` VALUES (5, 'lisi', 2, 0.000000, 'system', '2019-7-7 11:30:09', '2019-7-7 11:30:09', 'system');  
INSERT INTO `t_customer_credit` VALUES (6, 'lisi', 3, 0.000000, 'system', '2019-7-7 11:30:09', '2019-7-7 11:30:09', 'system');  
```

如果我们要一行显示用户的三个额度，而不是 3 条记录显示 3 个额度，我们应该怎么做，方式有很多种，这里提供如下 3 种

```sql
-- 1、最容易想到的IF，不具备移植性，不推荐  
SELECT login_name,  
    MAX(IF(credit_type=1, amount, 0)) freeAmount,  
    MAX(IF(credit_type=2, amount, 0)) freezeAmount,  
    MAX(IF(credit_type=3, amount, 0)) promotionAmount  
FROM t_customer_credit GROUP BY login_name;  
  
-- 2、CASE表达式，标准的 SQL 规范，具备移植性，推荐使用  
SELECT login_name,  
    MAX(CASE WHEN credit_type = 1 THEN amount ELSE 0 END) freeAmount,  
    MAX(CASE WHEN credit_type = 2 THEN amount ELSE 0 END) freezeAmount,  
    MAX(CASE WHEN credit_type = 3 THEN amount ELSE 0 END) promotionAmount  
FROM t_customer_credit GROUP BY login_name;  
  
-- 3、自连接，数据量大的情况下，结合索引，效率不错，具备移植性  
SELECT  
    a.login_name,a.amount freeAmount,  
    b.amount freezeAmount,  
    c.amount promotionAmount  
FROM (  
    SELECT login_name, amount FROM t_customer_credit WHERE credit_type = 1  
)a  
LEFT JOIN t_customer_credit b ON a.login_name = b.login_name AND b.credit_type = 2  
LEFT JOIN t_customer_credit c ON a.login_name = c.login_name AND c.credit_type = 3;  
```

![图片](https://mmbiz.qpic.cn/mmbiz_gif/Xruic8OIYw5vB6bvKh9ib6O6roWAicnfiaB0Da6n6k8RgnQAUfJ0jl5skB40iaGjJRz3aqsOvxb2L2JIYUSkO8icV4gg/640?wx_fmt=gif&wxfrom=5&wx_lazy=1)图片

无论是 IF 还是 CASE表达式，都结合了 GROUP BY 与聚合函数，效率是个问题，而自连接是效率最高的，不管在不在 `login_name` 上加索引

##### 转换统计

将已有编号方式转换为新的方式并统计，在进行非定制化统计时，我们经常会遇到将已有编号方式转换为另外一种便于分析的方式并进行统计的需求。假设我们有如下表

```sql
DROP TABLE t_province_population;  
CREATE TABLE t_province_population (  
  id tinyint(2) unsigned NOT NULL AUTO_INCREMENT COMMENT '自增主键',  
  province_name varchar(50) NOT NULL COMMENT '省份名',  
  sex tinyint(1) NOT NULL COMMENT '性别，1:男,2:女',  
  population int(11) NOT NULL COMMENT '人口数',  
  PRIMARY KEY (id)  
);  
  
INSERT INTO t_province_population(province_name,sex,population)  
VALUES  
("黑龙江", 1 ,20),  
("黑龙江", 2 ,18),  
("内蒙古", 1 ,7),  
("内蒙古", 2 ,8),  
("海南", 1 ,20),  
("海南", 2 ,22),  
("西藏", 1 ,8),  
("西藏", 2 ,7),  
("浙江", 1 ,35),  
("浙江", 2 ,35),  
("台湾", 1 ,26),  
("台湾", 2 ,23),  
("河南", 1 ,40),  
("河南", 2 ,38),  
("湖北", 1 ,27),  
("湖北", 2 ,24);  
  
SELECT * FROM t_province_population;  
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xruic8OIYw5vB6bvKh9ib6O6roWAicnfiaB0AK9MwwcC8EAGQCmvhagneGm6DhGxNmTJm7AOIML3vngZPcIsicH7a5w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)图片

我们需要按各个省所在的位置，统计出东南西北中，各个区域内的人口数量

东：浙江、台湾，西：西藏，南：海南，北：黑龙江、内蒙古，中：湖北、河南

可能有人觉得这个表设计的不合理，应该在设计之初就应该多加一个区域字段（district）来标明各省所属区域。最好的做法确实是这样，但这得需要我们在设计之初的时候能考虑得到，或者有这样的需求，假设我们设计之初没有这样的需求，而我们也没考虑到，那么有没有什么办法来实现了？我们可以这样来写 SQL

```sql
-- 通用写法，适用于多种数据库  
SELECT CASE province_name  
    WHEN '浙江' THEN '东'  
    WHEN '台湾' THEN '东'  
    WHEN '海南' THEN '南'  
    WHEN '西藏' THEN '西'  
    WHEN '黑龙江' THEN '北'  
    WHEN '内蒙古' THEN '北'  
    WHEN '河南' THEN '中'  
    WHEN '湖北' THEN '种'  
    ELSE '其他' END district,  
    SUM(population) populations  
FROM t_province_population  
GROUP BY CASE province_name  
    WHEN '浙江' THEN '东'  
    WHEN '台湾' THEN '东'  
    WHEN '海南' THEN '南'  
    WHEN '西藏' THEN '西'  
    WHEN '黑龙江' THEN '北'  
    WHEN '内蒙古' THEN '北'  
    WHEN '河南' THEN '中'  
    WHEN '湖北' THEN '中'  
    ELSE '其他' END;  
  
-- MySQL支持写法，移植性差  
SELECT CASE province_name  
    WHEN '浙江' THEN '东'  
    WHEN '台湾' THEN '东'  
    WHEN '海南' THEN '南'  
    WHEN '西藏' THEN '西'  
    WHEN '黑龙江' THEN '北'  
    WHEN '内蒙古' THEN '北'  
    WHEN '河南' THEN '中'  
    WHEN '湖北' THEN '中'  
    ELSE '其他' END district,  
    SUM(population) populations  
FROM t_province_population  
GROUP BY district;  
```

结果如下

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xruic8OIYw5vB6bvKh9ib6O6roWAicnfiaB0t7zzFOY2jdNd4N13PLo4yvv05Oib0o2p1kiaicMINvSa6ZC4gT2VbeF2g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

假设我们需要对各个省份做一个人口数级别的统计，统计出各个级别的数量

- level_1：`population < 20`
- level_2：`20 <= population < 50`
- level_3：`50 <= population < 70`
- level_4：`>= 70`；

统计出 level_1 ~ level_4 的数量各有多少

SQL 与执行结果如下

```sql
SELECT   
    CASE WHEN population < 20 THEN 'level_1'  
        WHEN population >= 20 AND population < 50 THEN 'level_2'  
        WHEN population >= 50 AND population < 70 THEN 'level_3'  
        WHEN population >= 70 THEN 'level_4'  
        ELSE NULL   
    END pop_level,  
    COUNT(*) cnt  
FROM (  
    SELECT province_name,SUM(population) population FROM t_province_population GROUP BY province_name  
)a  
GROUP BY   
    CASE WHEN population < 20 THEN 'level_1'  
        WHEN population >= 20 AND population < 50 THEN 'level_2'  
        WHEN population >= 50 AND population < 70 THEN 'level_3'  
        WHEN population >= 70 THEN 'level_4'  
        ELSE NULL   
    END;  
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xruic8OIYw5vB6bvKh9ib6O6roWAicnfiaB0LJDR46auczG5ZRJM0MqrIXj8MVagFdQtSF0Wak2peNmezRLtjluDhg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

这种转换统计还是比较常用的，重点就是 GROUP BY 子句的写法。

##### 条件分支

###### SELECT 条件分支

还是以上面的 `t_province_population` 为例，如果我们想要直观的知道各个省份的男、女数量情况，类似如下

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xruic8OIYw5vB6bvKh9ib6O6roWAicnfiaB0cJhVOYvsGj8HxagWLzIE3kr7JNlKhJTIicGrUnpITEyMS0BeK6esbdA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

我们要怎么写 SQL？有如下两种方法

```sql
-- 1、CASE表达式 集合 GROUP BY  
SELECT province_name,  
    SUM(CASE WHEN sex = 1 THEN population ELSE 0 END) c,  
    SUM(CASE WHEN sex = 2 THEN population ELSE 0 END) f_pops  
FROM t_province_population  
GROUP BY province_name;  
  
-- 2、自关联  
SELECT t.province_name, t.population m_pops, a.population f_pops  
FROM t_province_population t  
LEFT JOIN t_province_population a  
ON t.province_name = a.province_name  
WHERE t.sex = 1 AND a.sex = 2;  
```

其实就是行转列，行转列更容易懂

###### **UPDATE 条件分支**

我们有一张薪资表，如下

```sql
CREATE TABLE t_user_salaries(  
  id int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '自增主键',  
  name varchar(50) NOT NULL COMMENT '姓名',  
    sex tinyint(1) NOT NULL COMMENT '性别，1:男,2:女',  
  salary int(11) NOT NULL COMMENT '薪资',  
  PRIMARY KEY (id)  
);  
  
INSERT INTO t_user_salaries(name, sex,salary) VALUES  
("张三", 1, 30000),  
("李四", 1, 27000),  
("王五", 1, 22000),  
("菲菲", 2, 24000),  
("赵六", 1, 29000);  
  
SELECT * FROM t_user_salaries;  
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xruic8OIYw5vB6bvKh9ib6O6roWAicnfiaB0psISYMicQIJTge2wCyJ42PrmfFsR7iaUmdHJTrYLGKicS5tbYe9lwSheQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

假设现在需要根据以下条件对该表的数据进行更新：1、对当前工资为 30000 元以上的员工，降薪 10%，2、对当前工资为 25000 元以上且不满 28000 元的员工，加薪 20%。调整之后的薪资如下所示

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xruic8OIYw5vB6bvKh9ib6O6roWAicnfiaB0rcnicBeXZWBLyMaLYiabVibuaLqLebkG67WfVpzYpYQHMvcgrmfeRD53w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

乍一看，分别执行下面两个 UPDATE 操作好像就可以做到，但是我们执行下看看结果

```sql
-- 条件1  
UPDATE t_user_salaries  
SET salary = salary * 0.9  
WHERE salary >= 30000;  
  
-- 条件2  
UPDATE t_user_salaries  
SET salary = salary * 1.2  
WHERE salary >= 25000 AND salary < 28000;  
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xruic8OIYw5vB6bvKh9ib6O6roWAicnfiaB0AjoFpG4R6Fv1w6bSyQE4sQiaYLmHuhSXP7D6oRK05O52g2Hg9ya603Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

我们发现张三的薪资不降反升了！这是因为执行 条件1的SQL后，张三的薪资又满足条件2了，所以又更新了一遍，导致他的薪资变多了，有人可能会说，把条件1和条件2的SQL换下顺序不就好了吗，我们来试试

```sql
-- 条件2  
UPDATE t_user_salaries  
SET salary = salary * 1.2  
WHERE salary >= 25000 AND salary < 28000;  
  
-- 条件1  
UPDATE t_user_salaries  
SET salary = salary * 0.9  
WHERE salary >= 30000;  
```

![图片](https://mmbiz.qpic.cn/mmbiz_gif/Xruic8OIYw5vB6bvKh9ib6O6roWAicnfiaB012QLonkHEbW3lwujAu7aBpV3tQuLeiblFBO7TH8YrPjs6bEUIPgoYKw/640?wx_fmt=gif&wxfrom=5&wx_lazy=1)

张三的薪资是降对了，可李四的薪资却涨错了！这是因为李四的薪资满足条件2，升了 20% 之后又满足条件1，又降了 10%。难道就没有就没有正确的方式了？我们来看看这个 SQL

```sql
UPDATE t_user_salaries SET salary =   
    CASE WHEN salary >= 30000 THEN salary * 0.9  
            WHEN salary >= 25000 AND salary < 28000 THEN salary * 1.2  
            ELSE salary  
    END;  
  
SELECT * FROM t_user_salaries;  
```

![图片](https://mmbiz.qpic.cn/mmbiz_gif/Xruic8OIYw5vB6bvKh9ib6O6roWAicnfiaB09vEc5ica8dNp0AzBDib1gC06kSjicm3WfFewV9N6fHMVOD0cmzYP94vyQ/640?wx_fmt=gif&wxfrom=5&wx_lazy=1)

完美不？特别完美，这个技巧的应用范围很广，值得我们掌握

#### CHECK 约束

> 注意：CHECK 是标准的 SQL，但是 MySQL 却没有实现它，所以 CHECK 在 MySQL 中是不起作用的！

回到我们的薪资表，假设某个公司有这样一个无理的规定：女性员工的工资不得高于50000，我们如果实现它？ 方式有两种：1、代码层面控制 、2、数据库表加约束。

代码层面控制就不多说了，这我们平时最能想到的，实际也是用的最多的；那从表约束，我们该如何实现了，像这样吗？

```sql
-- 创建表的时候增加约束  
CREATE TABLE t_user_salaries_check(  
  name varchar(50) NOT NULL COMMENT '姓名',  
    sex tinyint(1) NOT NULL COMMENT '性别，1:男,2:女',  
  salary int(11) NOT NULL COMMENT '薪资',  
    CONSTRAINT chk_sex_salary CHECK (sex=2 AND salary <= 50000)  
);  
  
-- 若t_user_salaries_check已创建，则补充上约束  
ALTER TABLE t_user_salaries_check  
ADD CONSTRAINT chk_sex_salary CHECK (sex=2 AND salary <= 50000);  
```

这么实现你会发现公司的男同事都会提着刀来找你了，因为没有他们的薪资，这个约束会导致录入不了男性的薪资！因为我们的约束是：`sex=2 AND salary < = 50000 `表示 “是女性，并且薪资不能高于50000”，而不是：“如果是女性，薪资不高于50000”。正确的约束条件应该这么写

```sql
-- 创建表的时候增加约束  
CREATE TABLE t_user_salaries_check(  
  name varchar(50) NOT NULL COMMENT '姓名',  
    sex tinyint(1) NOT NULL COMMENT '性别，1:男,2:女',  
  salary int(11) NOT NULL COMMENT '薪资',  
  PRIMARY KEY (id),  
    CONSTRAINT chk_sex_salary CHECK(  
        CASE WHEN sex = 2 THEN   
                        CASE WHEN salary <= 50000 THEN 1   
                                ELSE 0   
                        END  
                ELSE 1   
        END = 1 )  
);  
  
-- 若t_user_salaries_check已创建，则补充上约束  
ALTER TABLE t_user_salaries_check  
ADD CONSTRAINT chk_sex_salary CHECK(  
    CASE WHEN sex = 2 THEN   
                        CASE WHEN salary <= 50000 THEN 1   
                                ELSE 0   
                        END  
                ELSE 1   
        END = 1   
);  
```

CASE表达式还有很多其他的用处，强大的不得了，而且高度灵活；用好它，能让我们写出更加契合的 SQL。

#### 总结

1、CASE表达式 是支撑 SQL 声明式编程的根基之一，也是灵活运用 SQL 时不可或缺的基础技能。作为表达式，CASE 表达式在执行时会被判定为一个固定值，因此它可以写在聚合函数内部；也正因为它是表达式，所以还可以写在SELECE 子句、GROUP BY 子句、WHERE 子句、ORDER BY 子句里。简单点说，在能写列名和常量的地方，通常都可以写 CASE 表达式

2、写 CASE表达式 的注意点

a、各个分支返回的数据类型要一致

b、养成写 ELSE 的好习惯

c、不要忘了写 END
