+++
categories = ["sql", "date", "time", "type", "json", "drill"]
type = "blog"
author = "Nathan Griffith"
date = "2015-12-03"
title = "Querying dates using SQL in a JSON document"

+++

In this article we're going to use Apache Drill to bend TIME ITSELF to our every whim. Or wait, not TIME ITSELF. I
meant, er... TIMESTAMPS. Don't worry&mdash;that's still pretty useful.

So let's say I have the results of a Twitter search in raw JSON and would like to know the range of dates that the
statuses span. For this we'll need the timestamp information, which is held in the 'created_at' key of each object.

Now take a look at my search results for the word 'squark.' (Yes, that's a real word. You can Google it and learn
some particle physics.) We'll first try to find the dates spanned by the results using this method:

```
> SELECT MIN(t.created_at) FROM dfs.`/path/to/search.json` t;
+---------------------------------+
|             EXPR$0              |
+---------------------------------+
| Fri Nov 27 02:34:52 +0000 2015  |
+---------------------------------+
```

```
> SELECT MAX(t.created_at) FROM dfs.`/path/to/search.json` t;
+---------------------------------+
|             EXPR$0              |
+---------------------------------+
| Wed Nov 25 19:42:49 +0000 2015  |
+---------------------------------+
```

Well, the minimum date is later than the maximum date, so it's safe to say that didn't work. Here's the reason:
'created_at' is being treated as a string typed variable and not as a timestamp. Fixing this is fairly straightforward,
we'll just have to use the `TO_TIMESTAMP()` function to convert the type.

`TO_TIMESTAMP()` takes two arguments: the first one is the name of the column you'd like to convert, and the second one
is a string that describes the format of the date that you're converting (use this
[page](http://joda-time.sourceforge.net/apidocs/org/joda/time/format/DateTimeFormat.html) as a reference).

So in my case I can convert my date strings to the timestamp type with these arguments:

```
TO_TIMESTAMP(t.created_at, 'EEE MMM dd HH:mm:ss +0000 yyyy')
```

And now let's try to find the first and last date again:

```
> SELECT MIN(TO_TIMESTAMP(t.created_at, 'EEE MMM dd HH:mm:ss +0000 yyyy')) FROM
> dfs.`/path/to/search.json` t;
+------------------------+
|         EXPR$0         |
+------------------------+
| 2015-11-25 19:42:49.0  |
+------------------------+
```

```
> SELECT MAX(TO_TIMESTAMP(t.created_at, 'EEE MMM dd HH:mm:ss +0000 yyyy')) FROM
> dfs.`/path/to/search.json` t;
+------------------------+
|         EXPR$0         |
+------------------------+
| 2015-12-02 14:42:30.0  |
+------------------------+
```

Much better!

Rigorously manipulating time data is the sort of fiddly work that can give even experienced technical types an anxious
rash, so it's nice to know that Drill has your back when it comes to making date format conversions.
