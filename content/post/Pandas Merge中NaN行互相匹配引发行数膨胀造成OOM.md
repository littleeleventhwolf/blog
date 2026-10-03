---
title: 'Pandas Merge中NaN行互相匹配引发行数膨胀造成OOM'
date: '2026-10-03T18:57:50+08:00'
lastmod: '2026-10-03T18:57:50+08:00'
keywords: ['pandas']
categories: ['pandas']
tags: ['pandas']
author: '小十一狼'
---

# 问题现象

前段时间排查了一类作业对内存需求特别大，经常发生 OOM。通过不断增大内存，并观察 py-spy 输出的调用栈，发现调用栈卡在一个逻辑上，逻辑大致如下：读取目录下所有 CSV 文件，解析后按某个字段进行 `pd.merge(on=...)`，连接方式是 `how='outer'`。

# 根因分析

问题链路如下：

1. 目录下存在非标准 CSV 文件；
2. `pd.read_csv` 解析异常文件时部分字段读取失败；
3. 用于 `merge` 的 key 列因此出现大量 `NaN`；
4. Pandas 的 merge 会把左右两边 key 都是 `NaN` 的行互相匹配；
5. 两边 `NaN` key 行数较多时，形成多对多膨胀；
6. 行数激增，内存失控，触发 OOM。

所以，总结下来：解析异常导致 `merge` key 列产生大量 `NaN`，而 Pandas `merge` 会把两边 key 都是 `NaN` 的行互相匹配，与 SQL 中 `NULL=NULL` 不匹配的行为不同。当两边都有大量 `NaN` key 时，`outer merge` 会放大行数。

我们来对比下 Pandas merge 和 SQL Join 的差异。

# 对照实验

用极简的数据分别在 Pandas 和 SQL 中执行相同语义的 Join，观察行数差异。

## 1. 造数据

```python
import pandas as pd

# 左表 17 行，key 全是 NaN
left = pd.DataFrame({
    "join_key": [pd.NA] * 17,
    "payload_left": range(17),
})

# 右表 17 行，key 也全是 NaN
right = pd.DataFrame({
    "join_key": [pd.NA] * 17,
    "payload_right": range(17, 34),
})
```

## 2. Pandas 结果

```python
merged = pd.merge(left, right, on="join_key", how="outer")
print(merged.shape) # (289, 3) -> 17 * 17
```

两边都是 `NaN` 时，Pandas 会把它们互相匹配，形成 17*17=289 行的笛卡尔积。

## 3. SQL 对照

```sql
$ sqlite3 /tmp/test.db
SQLite version 3.43.2 2023-10-10 13:08:14
Enter ".help" for usage hints.
sqlite> CREATE TABLE t_left (join_key TEXT, payload_left INT);
sqlite> CREATE TABLE t_right (join_key TEXT, payload_right INT);
sqlite> INSERT INTO t_left (join_key, payload_left) VALUES
   ...> (NULL,0),(NULL,1),(NULL,2),(NULL,3),(NULL,4),(NULL,5),(NULL,6),(NULL,7),(NULL,8),
   ...> (NULL,9),(NULL,10),(NULL,11),(NULL,12),(NULL,13),(NULL,14),(NULL,15),(NULL,16);
sqlite> INSERT INTO t_right (join_key, payload_right) VALUES
   ...> (NULL,17),(NULL,18),(NULL,19),(NULL,20),(NULL,21),(NULL,22),(NULL,23),(NULL,24),(NULL,25),
   ...> (NULL,26),(NULL,27),(NULL,28),(NULL,29),(NULL,30),(NULL,31),(NULL,32),(NULL,33);
sqlite> SELECT COUNT(*) AS outer_rows FROM t_left l FULL OUTER JOIN t_right r ON l.join_key = r.join_key;
34
```

可以看到有 34 个，不是笛卡尔积。如下展示全部数据：

```sql
sqlite> .headers on
sqlite> .mode table
sqlite> SELECT l.join_key, l.payload_left, r.payload_right FROM t_left l FULL OUTER JOIN t_right r ON l.join_key = r.join_key;
+----------+--------------+---------------+
| join_key | payload_left | payload_right |
+----------+--------------+---------------+
|          | 0            |               |
|          | 1            |               |
|          | 2            |               |
|          | 3            |               |
|          | 4            |               |
|          | 5            |               |
|          | 6            |               |
|          | 7            |               |
|          | 8            |               |
|          | 9            |               |
|          | 10           |               |
|          | 11           |               |
|          | 12           |               |
|          | 13           |               |
|          | 14           |               |
|          | 15           |               |
|          | 16           |               |
|          |              | 17            |
|          |              | 18            |
|          |              | 19            |
|          |              | 20            |
|          |              | 21            |
|          |              | 22            |
|          |              | 23            |
|          |              | 24            |
|          |              | 25            |
|          |              | 26            |
|          |              | 27            |
|          |              | 28            |
|          |              | 29            |
|          |              | 30            |
|          |              | 31            |
|          |              | 32            |
|          |              | 33            |
+----------+--------------+---------------+
```

# 规避与修复

既然 Pandas `merge` 在 on `NaN` 是笛卡尔积，那我们写代码的时候就要注意：不让 `NaN` key 进入 `merge`。

`merge` 前检查 key 列质量：

```python
for df, name in [(left, "left"), (right, "right")]:
    null_cnt = df["join_key"].isnull().sum();
    uniq_cnt = df["join_key"].nunique()
    print(f"{name}: nulls={null_cnt}, unique_keys={uniq_cnt}, rows={len(df)}")
```

如果 key 列存在大量 `NaN`，应先修复源头。常见处理方式：

```python
# 丢弃 key 为 NaN 的行
left = left.dropna(subset=['join_key'])
right = right.dropna(subset=['join_key'])
```
