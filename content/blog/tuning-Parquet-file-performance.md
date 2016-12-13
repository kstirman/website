+++
type = "blog"
author = "Nathan Griffith"
categories = ["parquet", "hdfs", "files", "drill", "performance"]
date = "2015-12-13"
title = "Tuning Parquet file performance"

+++

Today I'd like to pursue a brief discussion about how changing the size of a Parquet file's 'row group' to match a file
system's block size can effect the efficiency of read and write performance. This tweak can be especially important on
HDFS environments in which I/O is intrinsically tied to network operations. (Note: If you'd first like to learn about
Parquet in a less 'nuts and bolts' manner, let me recommend [this
post](http://www.dremio.com/blog/sql-and-parquet-a-simple-demo/) in which I provide a simple demo of the format using
Apache Drill).

To understand why the 'row group' size matters, it might be useful to first understand what the heck a 'row group' is.
For this let's refer to Figure 1, which is a simple illustration of the Parquet file format.

<br>
<br>
<p style="text-align: center;">
<img style="max-width: 100%;" src="/img/parquet_block1.png">
</p>
<p style="text-align: center; font-style: italic;">Figure 1: The basic anatomy of a Parquet file. The left file contains
one row group, while the right is comprised of several.</p>
<br>
<br>

As you can see, a row group is a segment of the Parquet file that holds serialized (and compressed!) arrays of column
entries. Since bigger row groups mean longer continuous arrays of column data (which is the whole point of Parquet!),
bigger row groups are generally good news if you want faster Parquet file operations.

But how does the block size of the disk come into play? Let's look at Figure 2, which explores three different Parquet
storage scenarios for an HDFS file system. In Scenario A, very large Parquet files are stored using large row groups.
The large row groups are good for executing efficient column-based manipulations, but the groups and files are prone to
spanning multiple disk blocks, which risks latency by invoking I/O operations. In Scenario B, small files are stored
using a single small row group. This mitigates the number of block crossings, but reduces the efficacy of Parquet's
columnar storage format.

<br>
<br>
<p style="text-align: center;">
<img style="max-width: 100%;" src="/img/parquet_block2.png">
</p>
<p style="text-align: center; font-style: italic;">Figure 2: Three different Parquet storage scenarios for an HDFS file
system.</p>
<br>
<br>

An ideal situation is demonstrated in Scenario C, in which one large Parquet file with one large row group is stored in
one large disk block. This minimizes I/O operations, while maximizing the length of the stored columns. The [official
Parquet documentation](https://parquet.apache.org/documentation/latest/) recommends a disk block/row group/file size of
512 to 1024 MB on HDFS.

In Apache Drill, you can change the row group size of the Parquet files it writes by using the `ALTER SYSTEM SET`
command on the `store.parquet.block-size` variable. For instance to set a row group size of 1 GB, you would enter:

```
ALTER SYSTEM SET `store.parquet.block-size` = 1073741824;
```

(Note: larger block sizes will also require more memory to manage.)

Parquet and Drill are already extremely well integrated when it comes to data access and storage&mdash;this tweak just
enhances an already potent symbiosis!
