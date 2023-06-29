---
title: 杂谈-使用SQL计算最长连续字串
date: 2023-06-29 20:21:39
categories: 杂谈
tags: 杂谈
excerpt: 使用SQL计算最长连续字串
---
最长连续子串是指在一个字符串中，找到最长的连续出现的字符子串。例如，在字符串 "aabbbbaaa" 中，最长连续子串是 "bbbb"。

滑动窗口自然可以轻松解决这个问题，不过用纯SQL很难实现滑动窗口

自然，如果愿意写UDF，自然可以UDF一把梭，不过维护UDF带来了额外的成本

然后我们找找有没有纯SQL的实现方案

SQL最常使用的手段是group, 把相同的元素聚合起来，于是我们考虑如何让 连续的子串 中的 每一个字符 变为连续的值

我们可以使用差分手法，计算每个字符在该类型字符中的排名以及在整个字符串的排名

以aabbbbaaa为例：

| - | a | a | b | b | b | b | a | a | a |
|---|---|---|---|---|---|---|---|---|---|
|在该类型值中的排名| 1 | 2 | 1 | 2 | 3 | 4 | 3 | 4 | 5 |
|在整个字符串的排名| 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |

对于一个区间(假设区间第一个字符在该类型值中的排名为x1，在整个字符串的排名为x2), 如果都是同样的字符，那么：

区间第i个字符在该类型值中的排名为: x1+i, 在整个字符串的排名为: x2+i, 相减以后：(x2+i)-(x1+i)=(x2-x1)

也就是同一个区间如果都是同样的字符，那么两个排名相减以后是一样的值

这也表明，同样的字符如果两个排名相减以后是一样的，那么他们之间不会有其余字符

因为整个字符串的排名是依次增加的，不妨设为x2位与x2+i位，其中x2位在该类型值中的排名为: x1，如果之间有其余字符，那么x2+i位在该类型值中的排名必然小于x1+i, 与两个排名相减以后是一样的矛盾

至于不同字符两个排名相减会相同，这个不重要

这两个排名都可以利用窗口函数ROW_NUMBER计算，计算排名差分以后通过group可以得到一系列的连续的子串，就可以继续计算最长连续字串了

参考代码：


{% spoiler "表结构" %}

表结构：id 代表序列 value 代表值
|id|value|
|--|--|
|1|a|
|2|a|
|3|b|
|4|b|
|5|b|
|6|b|
|7|a|
|8|a|
|9|a|

{% endspoiler %}

```sql
WITH ordered_sequences AS (
  SELECT id, value,
         ROW_NUMBER() OVER (ORDER BY id) AS row_num,
         ROW_NUMBER() OVER (PARTITION BY value ORDER BY id) AS group_num
  FROM sequences
),
group_lengths AS (
  SELECT value, row_num - group_num as offset, COUNT(*) AS group_length
  FROM ordered_sequences
  GROUP BY 1, 2
),
max_group_length AS (
  SELECT value, MAX(group_length) AS max_length
  FROM group_lengths
  GROUP BY 1
)
SELECT *
FROM max_group_length
ORDER BY max_length DESC
LIMIT 1;
```

如果是想计算aabbbbaaa这样的形式的

|value|
|---|
|aabbbbaaa|

emm 将一行数据中的字符串拆分为多行不就好了？例如presto中使用UNSET