+++
title = "Reddit hates George Bush more than Vladimir Putin"
categories = ["drill", "sql", "udfs", "custom functions", "sentiment", "analysis", "reddit"]
type = "blog"
author = "Nathan Griffith"
date = "2016-01-29"
+++

Just as I promised, today I'm going to show off that nifty sentiment analysis UDF for Apache Drill that I discussed in
[the last article](http://www.dremio.com/blog/writing-a-custom-sql-function-for-sentiment-analysis/).

Today's data is [once
again](http://www.dremio.com/blog/old-and-busted-teasing-formerly-fashionable-websites-from-reddit-data/) provided by
this [awesome dump of Reddit
submissions](https://www.reddit.com/r/datasets/comments/3mg812/full_reddit_submission_corpus_now_available_2006/) that
date from 2006 up through last summer. Basically I just ran the sentiment analyzer function through submission titles,
examining a selection of politicians that I thought Reddit might feel strongly about.

In terms of nitty-gritty Drill stuff, I started by first making a view for the data that includes the sentiment score as
computed by the star of yesterday's post, the `SIMPLESENT()` function:

```
> USE dfs.tmp;
> CREATE VIEW testview AS SELECT LOWER(title) title, TO_TIMESTAMP(CAST(created_utc AS INT)) created, score, SIMPLESENT(title) sentiment FROM hdfs.`/data/RS_full_corpus.json`;
```

And from there I just computed a simple average of those scores over the entire corpus:

```
> SELECT AVG(sentiment) FROM testview WHERE title LIKE '%donald trump%';
+------------------------+
|         EXPR$0         |
+------------------------+
| -0.052544239386344636  |
+------------------------+
```

Remember negative scores indicate negative feelings, and likewise for positive scores. The somewhat surprising results
are compiled below in Figure 1.

<p style="text-align: center;">
<img style="max-width: 100%;" src="/img/reddit_politicians.png">
</p>
<p style="text-align: center; font-style: italic;"><b>Figure 1</b>: Sentiment analysis for various politicians based on
Reddit submission title.</p>
<br>
<br>
<br>
<br>

So, yes, according to this sentiment analysis, Reddit definitely dislikes George Bush more than Vladimir Putin. I always
think of Reddit as overall leaning a bit left in terms of politics, so it didn't shock me to see Barack Obama and Bernie
Sanders show up with positive values. However, it *did* surprise me to see Hillary Clinton score negatively. And not
only that, she scored even *more* negatively than Sarah Palin!

Finally, those of us who were frequent redditors during the 2008 election season will be far from confused by the
average sentiment score achieved by then nominal-Republican Ron Paul.

Reddit loves that guy.
