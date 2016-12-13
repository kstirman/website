+++
categories = ["sql", "parquet", "files", "drill"]
type = "blog"
author = "Nathan Griffith"
date = "2015-12-01"
title = "SQL and Parquet: A simple demo"

+++

For data wranglers wishing to store large amounts of text or numeric entries in an efficient fashion, the Apache Parquet
file format makes for an obvious choice. As a storage medium, Parquet has two important benefits afforded by its
columnar structure: 1.) Access to data columns can be made on an 'as needed' basis, increasing the overall speed of
queries, and 2.) Since all values in a column are serialized and compressed together, Parquet files take up much less
room than similar plain text or row-wise compressed files.

In this article I'll run through a simple SQL manipulation enabled by Apache Drill which results in the creation of data
stored in Parquet format. Drill's support for Parquet runs deep, and it's the default storage format for files created
as a result of a CTAS (CREATE TABLE... AS) command.

We'll start by downloading some parking data from the city of Aarhus, Denmark (available via [this
site](http://iot.ee.surrey.ac.uk:8080/datasets.html)), renaming the file so that it has a '.csvh' extention (as per
[this article](http://www.dremio.com/blog/sql-queries-on-csv-files-with-column-headers/)). Then we'll start the Drill
prompt with the usual incantation (`drill-embedded` for single-machine or `drill-conf` on a cluster) and do a simple
`SELECT *` to look at the data:

```
> SELECT * FROM dfs.`/path/to/aarhus_parking.csvh` LIMIT 10;
+---------------+--------------------------+------+--------------+----------------+----------------------+
| vehiclecount  |        updatetime        | _id  | totalspaces  |   garagecode   |      streamtime      |
+---------------+--------------------------+------+--------------+----------------+----------------------+
| 0             | 2014-05-22 09:09:04.145  | 1    | 65           | NORREPORT      | 2014-11-03 16:18:44  |
| 0             | 2014-05-22 09:09:04.145  | 2    | 512          | SKOLEBAKKEN    | 2014-11-03 16:18:44  |
| 869           | 2014-05-22 09:09:04.145  | 3    | 1240         | SCANDCENTER    | 2014-11-03 16:18:44  |
| 22            | 2014-05-22 09:09:04.145  | 4    | 953          | BRUUNS         | 2014-11-03 16:18:44  |
| 124           | 2014-05-22 09:09:04.145  | 5    | 130          | BUSGADEHUSET   | 2014-11-03 16:18:44  |
| 106           | 2014-05-22 09:09:04.145  | 6    | 400          | MAGASIN        | 2014-11-03 16:18:44  |
| 115           | 2014-05-22 09:09:04.145  | 7    | 210          | KALKVAERKSVEJ  | 2014-11-03 16:18:44  |
| 233           | 2014-05-22 09:09:04.145  | 8    | 700          | SALLING        | 2014-11-03 16:18:44  |
| 0             | 2014-05-22 09:39:01.803  | 9    | 65           | NORREPORT      | 2014-11-03 16:18:44  |
| 0             | 2014-05-22 09:39:01.803  | 10   | 512          | SKOLEBAKKEN    | 2014-11-03 16:18:44  |
+---------------+--------------------------+------+--------------+----------------+----------------------+
```

Before we save this table as a Parquet file, we need to switch to a writable workspace inside Drill. The command

```
USE dfs.tmp;
```

will do the trick&mdash;it tells Drill to write files to the system's /tmp directory. With that done all we need to do
is run a CTAS

```
CREATE TABLE parking AS SELECT * FROM dfs.`/path/to/aarhus_parking.csvh`;
```

and then rename the resulting temporary file:

```
mv /tmp/parking/0_0_0.parquet /path/to/parking.parquet
```

Now we can query the new Parquet file with:

```
SELECT * FROM dfs.`/path/to/parking.parquet` LIMIT 10;
```

which yields the same results as before. But what about file size? How does the compressed Parquet file compare to the
original CSV? Pretty darn favorably, as it turns out:

```
$ ls -lrth
total 8488
-rw-r--r--  1 user  staff   594K Nov 30 15:06 parking.parquet
-rw-r-----@ 1 user  staff   3.6M Nov 30 15:07 aarhus_parking.csvh
```

In round numbers, it's about 1/6th the size!

So that's an extremely brief tour of how easily Drill interfaces with the Parquet file format. Since the two are an
extremely popular combination, expect to see some more in-depth articles about using Drill with Parquet on the Dremio
Blog in the near future.
