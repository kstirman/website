+++
type = "blog"
author = "Nathan Griffith"
categories = ["twitter", "statistics", "functions", "drill"]
date = "2015-12-03"
title = "Performing basic statistics operations on Twitter data"

+++

I've used Drill's `AVG()` function quite a bit previously, but in this post I'd like to point out some other basic
statistics operations you can do in Drill.

For my trial data I'm going to reference my trusty corpus of nearly 52,000 tweets pulled from Twitter's streaming API,
and with it I'm going to try to get a sense of how long people prefer to make their Twitter profile's 'real name.' To do
this I'll take an average, a standard deviation, and a median of the length of this field in the data.

We'll start with the old news, and take an average:

```
> SELECT AVG(LENGTH(t.`user`.name)) namelength FROM dfs.`/path/to/tweets.small.json` t;
+--------------------+
|     namelength     |
+--------------------+
| 9.995608290315124  |
+--------------------+
```

Nothing to it, right? Drill also has a built-in function taking the standard deviation, which works in exactly the same way:

```
> SELECT STDDEV(LENGTH(t.`user`.name)) namelength FROM dfs.`/path/to/tweets.small.json` t;
+--------------------+
|     namelength     |
+--------------------+
| 4.981551245459341  |
+--------------------+
```

There's no built-in function for finding the median, but we can work it out by issuing a couple queries. First we'll use
a `COUNT()` to return the total number of rows, and then use that result to pull a value from the middle of the sorted
set of data.

```
> SELECT COUNT(*) FROM dfs.`/path/to/tweets.small.json` t;
+---------+
| EXPR$0  |
+---------+
| 51916   |
+---------+
```

```
> SELECT LENGTH(t.`user`.name) namelength FROM dfs.`/path/to/tweets.small.json` t ORDER BY namelength LIMIT 1 OFFSET 25958;
+-------------+
| namelength  |
+-------------+
| 10          |
+-------------+
```

Applying these statistical methods to found data via Drill is an easy task, and it's part of what makes it an attractive
piece of software for those wanting to efficiently explore new sets of information.
