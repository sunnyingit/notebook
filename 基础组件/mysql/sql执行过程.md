SQL执行过程

## SQL执行顺序

QL 语句有一个让大部分人都感到困惑的特性，就是：SQL语句的执行顺序跟其语句的语法顺序并不一致，错误顺序为：
1.SELECT
2. DISTINCT
3. FROM
4. WHERE
5. GROUP BY
6. HAVING
7. UNION
8. ORDER BY

实际顺序为：
1. FROM
2. WHERE
3. GROUP BY
4. HAVING
5. SELECT
6. DISTINCT
7. UNION
8. ORDER BY

ELECT 是在大部分语句执行了之后才执行的，严格的说是在 FROM 和 GROUP BY 之后执行的。理解这一点是非常重要的，这就是你不能在 WHERE 中使用在 SELECT 中设定别名的字段作为判断条件的原因。

比如：
、、、
SELECT A.x + A.y AS z FROM A WHERE z = 10
、、、

“z” 在此处不可用，因为SELECT是最后执行的语句！

select只是从一堆已经选好的数据里面拿自己需要的，理解这点非常重要。
