## 1. SQL

#### 1.1 介绍一下数据库分页

**参考答案**

MySQL的分页语法：

在MySQL中，SELECT语句默认返回所有匹配的行，它们可能是指定表中的每个行。为了返回第一行或前几行，可使用LIMIT子句，以实现分页查询。LIMIT子句的语法如下：

```mysql
-- 在所有的查询结果中，返回前5行记录。
SELECT prod_name FROM products LIMIT 5;
-- 在所有的查询结果中，从第5行开始，返回5行记录。
SELECT prod_name FROM products LIMIT 5,5;
```

总之，带一个值的LIMIT总是从第一行开始，给出的数为返回的行数。带两个值的LIMIT可以指定从行号为第一个值的位置开始。

优化LIMIT分页：

在偏移量非常大的时候，例如 `LIMIT 10000,20` 这样的查询，这时MySQL需要查询10020条记录然后只返回最后20条，前面的10000条记录都将被抛弃，这样的代价是非常高的。如果所有的页面被访问的频率都相同，那么这样的查询平均需要访问半个表的数据。要优化这种查询，要么是在页面中限制分页的数量，要么是优化大偏移量的性能。

优化此类分页查询的一个最简单的办法就是尽可能地使用索引覆盖扫描，而不是查询所有的列，然后根据需要做一次关联操作再返回所需的列。对于偏移量很大的时候，这样做的效率会提升非常大。考虑下面的查询：

```mysql
SELECT film_id,description FROM sakila.film ORDER BY title LIMIT 50,5;
```

如果这个表非常大，那么这个查询最好改写成下面的样子：

```mysql
SELECT film.film_id,film.description 
FROM sakila.film
INNER JOIN (
    SELECT film_id FROM sakila.film ORDER BY title LIMIT 50,5
) AS lim USING(film_id);
```

这里的“延迟关联”将大大提升查询效率，它让MySQL扫描尽可能少的页面，获取需要访问的记录后再根据关联列回原表查询需要的所有列。这个技术也可以用于优化关联查询中的LIMIT子句。

有时候也可以将LIMIT查询转换为已知位置的查询，让MySQL通过范围扫描获得对应的结果。例如，如果在一个位置列上有索引，并且预先计算出了边界值，上面的查询就可以改写为：

```mysql
SELECT film_id,description FROM skila.film
WHERE position BETWEEN 50 AND 54 ORDER BY position;
```

对数据进行排名的问题也与此类似，但往往还会同时和GROUP BY混合使用，在这种情况下通常都需要预先计算并存储排名信息。

LIMIT和OFFSET的问题，其实是OFFSET的问题，它会导致MySQL扫描大量不需要的行然后再抛弃掉。如果可以使用书签记录上次取数的位置，那么下次就可以直接从该书签记录的位置开始扫描，这样就可以避免使用OFFSET。例如，若需要按照租赁记录做翻页，那么可以根据最新一条租赁记录向后追溯，这种做法可行是因为租赁记录的主键是单调增长的。首先使用下面的查询获得第一组结果：

```mysql
SELECT * FROM sakila.rental ORDER BY rental_id DESC LIMIT 20;
```

假设上面的查询返回的是主键16049到16030的租赁记录，那么下一页查询就可以从16030这个点开始：

```mysql
SELECT * FROM sakila.rental 
WHERE rental_id < 16030 ORDER BY rental_id DESC LIMIT 20;
```

该技术的好处是无论翻页到多么后面，其性能都会很好。

#### 1.2 介绍一下SQL中的聚合函数

**参考答案**

常用的聚合函数有COUNT()、AVG()、SUM()、MAX()、MIN()，下面以MySQL为例，说明这些函数的作用。

COUNT()函数：

COUNT()函数统计数据表中包含的记录行的总数，或者根据查询结果返回列中包含的数据行数，它有两种用法：

- COUNT(*)计算表中总的行数，不管某列是否有数值或者为空值。
- COUNT(字段名)计算指定列下总的行数，计算时将忽略空值的行。

COUNT()函数可以与GROUP BY一起使用来计算每个分组的总和。

AVG()函数()：

AVG()函数通过计算返回的行数和每一行数据的和，求得指定列数据的平均值。

AVG()函数可以与GROUP BY一起使用，来计算每个分组的平均值。

SUM()函数：

SUM()是一个求总和的函数，返回指定列值的总和。

SUM()可以与GROUP BY一起使用，来计算每个分组的总和。

MAX()函数：

MAX()返回指定列中的最大值。

MAX()也可以和GROUP BY关键字一起使用，求每个分组中的最大值。

MAX()函数不仅适用于查找数值类型，也可应用于字符类型。

MIN()函数：

MIN()返回查询列中的最小值。

MIN()也可以和GROUP BY关键字一起使用，求出每个分组中的最小值。

MIN()函数与MAX()函数类似，不仅适用于查找数值类型，也可应用于字符类型。

#### 1.3 表跟表是怎么关联的？

**参考答案**

表与表之间常用的关联方式有两种：内连接、外连接，下面以MySQL为例来说明这两种连接方式。

内连接：

内连接通过INNER JOIN来实现，它将返回两张表中满足连接条件的数据，不满足条件的数据不会查询出来。

外连接：

外连接通过OUTER JOIN来实现，它会返回两张表中满足连接条件的数据，同时返回不满足连接条件的数据。外连接有两种形式：左外连接（LEFT OUTER JOIN）、右外连接（RIGHT OUTER JOIN）。

- 左外连接：可以简称为左连接（LEFT JOIN），它会返回左表中的所有记录和右表中满足连接条件的记录。
- 右外连接：可以简称为右连接（RIGHT JOIN），它会返回右表中的所有记录和左表中满足连接条件的记录。

除此之外，还有一种常见的连接方式：等值连接。这种连接是通过WHERE子句中的条件，将两张表连接在一起，它的实际效果等同于内连接。出于语义清晰的考虑，一般更建议使用内连接，而不是等值连接。

以上是从语法上来说明表与表之间关联的实现方式，而从表的关系上来说，比较常见的关联关系有：一对多关联、多对多关联、自关联。

- 一对多关联：这种关联形式最为常见，一般是两张表具有主从关系，并且以主表的主键关联从表的外键来实现这种关联关系。另外，以从表的角度来看，它们是具有多对一关系的，所以不再赘述多对一关联了。
- 多对多关联：这种关联关系比较复杂，如果两张表具有多对多的关系，那么它们之间需要有一张中间表来作为衔接，以实现这种关联关系。这个中间表要设计两列，分别存储那两张表的主键。因此，这两张表中的任何一方，都与中间表形成了一对多关系，从而在这个中间表上建立起了多对多关系。
- 自关联：自关联就是一张表自己与自己相关联，为了避免表名的冲突，需要在关联时通过别名将它们当做两张表来看待。一般在表中数据具有层级（树状）时，可以采用自关联一次性查询出多层级的数据。

#### 1.4 说一说你对外连接的了解

**参考答案**

外连接通过OUTER JOIN来实现，它会返回两张表中满足连接条件的数据，同时返回不满足连接条件的数据。常见的外连接有两种形式：左外连接（LEFT OUTER JOIN）、右外连接（RIGHT OUTER JOIN）。

- 左外连接：可以简称为左连接（LEFT JOIN），它会返回左表中的所有记录和右表中满足连接条件的记录。
- 右外连接：可以简称为右连接（RIGHT JOIN），它会返回右表中的所有记录和左表中满足连接条件的记录。

实际上，外连接还有一种形式：完全外连接（FULL OUTER JOIN），但MySQL不支持这种形式。

#### 1.5 说一说数据库的左连接和右连接

**参考答案**

外连接通过OUTER JOIN来实现，它会返回两张表中满足连接条件的数据，同时返回不满足连接条件的数据。常见的外连接有两种形式：左外连接（LEFT OUTER JOIN）、右外连接（RIGHT OUTER JOIN）。

- 左外连接：可以简称为左连接（LEFT JOIN），它会返回左表中的所有记录和右表中满足连接条件的记录。
- 右外连接：可以简称为右连接（RIGHT JOIN），它会返回右表中的所有记录和左表中满足连接条件的记录。

#### 1.6 SQL中怎么将行转成列？

**参考答案**

我们以MySQL数据库为例，来说明行转列的实现方式。

首先，假设我们有一张分数表（tb_score），表中的数据如下图：

![img](https://static.nowcoder.com/images/activity/2021jxy/java/img/sql-1.png)

然后，我们再来看一下转换之后需要得到的结果，如下图：

![img](https://static.nowcoder.com/images/activity/2021jxy/java/img/sql-2.png)

可以看出，这里行转列是将原来的subject字段的多行内容选出来，作为结果集中的不同列，并根据userid进行分组显示对应的score。通常，我们有两种方式来实现这种转换。

1. 使用 `CASE...WHEN...THEN` 语句实现行转列，参考如下代码：

   ```mysql
   SELECT userid,
   SUM(CASE `subject` WHEN '语文' THEN score ELSE 0 END) as '语文',
   SUM(CASE `subject` WHEN '数学' THEN score ELSE 0 END) as '数学',
   SUM(CASE `subject` WHEN '英语' THEN score ELSE 0 END) as '英语',
   SUM(CASE `subject` WHEN '政治' THEN score ELSE 0 END) as '政治' 
   FROM tb_score 
   GROUP BY userid
   ```

   注意，SUM() 是为了能够使用GROUP BY根据userid进行分组，因为每一个userid对应的subject="语文"的记录只有一条，所以SUM() 的值就等于对应那一条记录的score的值。假如userid ='001' and subject='语文' 的记录有两条，则此时SUM() 的值将会是这两条记录的和，同理，使用Max()的值将会是这两条记录里面值最大的一个。但是正常情况下，一个user对应一个subject只有一个分数，因此可以使用SUM()、MAX()、MIN()、AVG()等聚合函数都可以达到行转列的效果。

2. 使用 `IF()` 函数实现行转列，参考如下代码：

   ```mysql
   SELECT userid,
   SUM(IF(`subject`='语文',score,0)) as '语文',
   SUM(IF(`subject`='数学',score,0)) as '数学',
   SUM(IF(`subject`='英语',score,0)) as '英语',
   SUM(IF(`subject`='政治',score,0)) as '政治' 
   FROM tb_score 
   GROUP BY userid
   ```

   注意，`IF(subject='语文',score,0)` 作为条件，即对所有subject='语文'的记录的score字段进行SUM()、MAX()、MIN()、AVG()操作，如果score没有值则默认为0。