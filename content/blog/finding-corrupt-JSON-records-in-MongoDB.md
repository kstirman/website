+++
categories = ["drill", "mongodb", "JSON", "debugging", "troubleshooting"]
type = "blog"
author = "Nathan Griffith"
date = "2015-11-30"
title = "Finding corrupt JSON records in MongoDB"

+++

A few weeks ago I mangled a MongoDB collection I was working with by importing a file I'd corrupted with a manual edit.
The change I introduced was small, but serious enough to cause my Drill queries to fail. In this post I'm going to
simulate this situation with an intentionally besmirched JSON file imported into MongoDB.

Today's data set comes from some tweets that I pulled from Twitter's streaming API. So let's try to use Drill to query
for some tweet id's:

```
> SELECT id FROM mongo.tweets.small;
Error: SYSTEM ERROR: IllegalArgumentException: You tried to write a VarChar type when you are using a ValueWriter of
type NullableBigIntWriterImpl.
```

Annnnd: No luck!

But can I at least look at the first 10 values?

```
> SELECT id FROM mongo.tweets.small LIMIT 10;
Error: SYSTEM ERROR: IllegalArgumentException: You tried to write a VarChar type when you are using a ValueWriter of
type NullableBigIntWriterImpl.
```

Still nothing. But let's take a close look at the error message&mdash;it looks like Drill is upset about about variable
types. This is a good time to switch on Drill's `union_type` functionality, which will allow us to query a column that
has multiple types of data.

```
> ALTER SYSTEM SET `exec.enable_union_type` = true;
```

Now let's ask for the 'tweet id' value again, and this time tack on a column that tells us the type of the variable.

```
> SELECT id, TYPEOF(id) type FROM mongo.tweets.small LIMIT 10;
+---------------------+---------+
|         id          |  type   |
+---------------------+---------+
| 663858171623030785  | BIGINT  |
| 663858171635589120  | BIGINT  |
| 663858171627245572  | BIGINT  |
| 663858171631374336  | BIGINT  |
| 663858171631415296  | BIGINT  |
| 663858171648184320  | BIGINT  |
| 663858171635625984  | BIGINT  |
| 663858171623043072  | BIGINT  |
| 663858171618791424  | BIGINT  |
| 663858171627175936  | BIGINT  |
+---------------------+---------+
```

Looks like 'BIGINT' is the standard type for the 'id' values. So is there any case in which the type is *not* equal to
'BIGINT'? Let's use this query:

```
> SELECT id, TYPEOF(id) type FROM mongo.tweets.small WHERE TYPEOF(id) NOT LIKE 'BIGINT';
+---------------------+----------+
|         id          |   type   |
+---------------------+----------+
| 663858171627352065  | VARCHAR  |
+---------------------+----------+
```

Gotcha! I now have the offending tweet id in hand, so if I want to I can go back and fix the JSON file I imported. (In
this case I had put quotes around the value, which caused it to read in as a string and not a number. Removing those
quotes will do the trick and make the file read correctly.)

This makes for a fairly quick and easy way to hunt down issues in corrupted databases. Try it out next time you find
yourself struggling with something that looks like a type error.
