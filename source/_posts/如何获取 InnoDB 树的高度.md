---
title: 如何获取 InnoDB 树的高度
date: 2019/9/12 16:17:0
tags: [数据库]
mathjax: true
categories: [数据库]
---
磁盘存储数据的最小单元是扇区，一般一个扇区的大小是 512 字节，计算机操作系统为了更好地与磁盘通信，抽象出了簇或块的概念，一般大小为 4K。InnoDB 存储引擎也有自己的最小存储单元——页，默认页的大小为 16K，我们可以通过以下命令查看该值。

<!--more-->

```bash
mysql> show variables like 'innodb_page_size';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| innodb_page_size | 16384 |
+------------------+-------+
```

InnoDB 中所有的表空间文件的大小都是 16K 的整数倍。在 InnoDB 的表空间文件（这里说的不是共享的表空间，而是每个表独立的表空间）中，页号为 3 的页是主键索引的根页，其它索引的页号为 4，该值可以通过以下命令查看。

```bash
mysql> SELECT
b.name, a.name, index_id, type, a.space, a.PAGE_NO
FROM
information_schema.INNODB_SYS_INDEXES a,
information_schema.INNODB_SYS_TABLES b
WHERE
a.table_id = b.table_id AND a.space <> 0;
+--------------------------------------------+---------------------------------------+----------+------+-------+---------+
| name                                       | name                                  | index_id | type | space | PAGE_NO |
+--------------------------------------------+---------------------------------------+----------+------+-------+---------+
| demo/t_certification                       | PRIMARY                               |       79 |    3 |    48 |       3 |
| demo/t_certification                       | t_certification_idx                   |       80 |    0 |    48 |       4 |
| demo/t_company_abstract                    | PRIMARY                               |       72 |    3 |    43 |       3 |
| demo/t_company_location                    | PRIMARY                               |       73 |    3 |    44 |       3 |
| demo/t_copyrights_copyright                | PRIMARY                               |       81 |    3 |    49 |       3 |
+--------------------------------------------+---------------------------------------+----------+------+-------+---------+
```

因为主键索引 B+树的根页在整个表空间文件中的第 3 页，所以我们可以计算出它在文件中的偏移量为：`16384 byte * 3 = 49152 byte`。根据《MySQL技术内幕：InnoDB存储引擎》的描述，MySQL 的数据页中，前 38 个字节为 File Header，紧接着的 56 个字节为 Page Header。在 Page Header 中，第 27 到 28 两个字节为 PAGE_LEVEL，代表当前页在索引树中的位置，值 0x00 表示当前页为叶子节点，**叶子节点总是在第 0 层**。由此我们可以推断出，只要拿到主键索引的根页中 PAGE_LEVEL 的值就可以计算整棵树的高度。具体来说就是根页中第 65 到 66 这两个字节的值。我们首先计算整体的偏移量为：`49152 byte + 64 byte = 49216 byte`。接下来我们使用 `hexdump` 查看表空间文件指定偏移量上的数据。

```bash
# 默认使用两字节十六进制表示，-C 采用单字节十六进制
> hexdump -C -s 49216 -n 10 ../data/demo/t_eci_company.ibd
0000c040  00 03 00 00 00 00 00 00  00 29
0000c04a
> hexdump -C -s 49216 -n 10 ../data/demo/t_eci_count.ibd
0000c040  00 02 00 00 00 00 00 00  00 4a
0000c04a
```

可以看到例子中表 `t_eci_company` 的 PAGE_LEVEL 为 3，`t_eci_count` 的 PAGE_LEVEL 为 2，所以它们的树高分别为 4 和 3。

除了手工分析，我们还可以借助于工具。[innodb_ruby](https://github.com/jeremycole/innodb_ruby) 就是一款可以分析 innoDB 文件的工具，使用 ruby 编写。在安装完 ruby 后，使用 ruby 的包管理工具直接下载该工具，然后我们简单使用一下，可以看到它的结果与我们的分析是一致的。

```bash
> innodb_space -f ../data/demo/t_eci_company.ibd space-index-pages-summary | head -n 10
page        index   level   data    free    records
3           41      3       92      16160   2
4           41      0       14339   1909    16
5           41      0       15117   1131    16
6           41      0       8617    7633    9
7           41      0       12208   4042    12
8           41      0       9232    7018    9
9           41      0       12755   3495    13
10          41      0       7846    8404    8
11          41      0       14079   2169    15
```

# 参考
> [MySQL Internals Manual: High-Altitude View](https://dev.mysql.com/doc/internals/en/innodb-page-overview.html)