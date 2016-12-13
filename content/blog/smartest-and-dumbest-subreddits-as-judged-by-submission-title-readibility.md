+++
type = "blog"
author = "Nathan Griffith"
categories = ["drill", "sql", "udfs", "text", "readability", "reddit", "custom functions"]
date = "2016-02-06"
title = "Smartest and dumbest subreddits as judged by submission title readability"
+++

Alright, time to put the readability UDF from [my last 
post](http://www.dremio.com/blog/querying-for-reading-level-with-a-simple-udf/) to work on some data! For today's
analysis, I'll once again use this [Reddit submission
corpus](https://www.reddit.com/r/datasets/comments/3mg812/full_reddit_submission_corpus_now_available_2006/), which
contains submission data from from the years 2006-2015.

The questions that motivate today's analysis are simple, but fun: Which popular subreddits have the highest average
submission title reading level? Which ones have the lowest? Or, more glibly, "Which subreddits are smart, and which are
dumb?" My custom `READABILITY()` function for Drill will help us settle this.

As usual when performing a slightly sophisticated analysis, I first shuffle the data through some VIEWs in order to
grapple with it on my terms. In this case the prep work consisted of two VIEWs, which were constructed as follows:

```
CREATE VIEW reddit_readability AS
     SELECT title, READABILITY(title) ARI, subreddit
       FROM hdfs.`/data/RS_full_corpus.json`
      WHERE over_18 = 'false';
```

which is in turn fed into:

```
CREATE VIEW reddit AS
     SELECT subreddit, COUNT(title) posts, AVG(ARI) `avg ARI`
       FROM reddit_readability
      GROUP BY subreddit;
```

From here the query to ask for 'smart' subreddits (as I've defined them) is easy:

```
> SELECT subreddit, posts, `avg ARI` FROM reddit WHERE posts > 100000 ORDER BY `avg ARI` DESC LIMIT 10;
+------------------------+---------+---------------------+
|       subreddit        |  posts  |       avg ARI       |
+------------------------+---------+---------------------+
| spam                   | 287305  | 15.658034632014635  |
| longtail               | 134202  | 15.38567146310872   |
| ModerationLog          | 356311  | 12.622485326455028  |
| modlog                 | 266188  | 12.477493297559754  |
| RisingThreads          | 142485  | 11.752936255500762  |
| worldpolitics          | 163290  | 11.633828401867433  |
| WritingPrompts         | 150776  | 11.348865875111446  |
| environment            | 185858  | 11.167274719418218  |
| Random_Acts_Of_Amazon  | 184420  | 11.14375363175314   |
| conspiro               | 179705  | 10.868244970017907  |
+------------------------+---------+---------------------+
```

These results aren't too surprising. Once the internal Reddit stuff is out of the way you're left with some subjects
that are pretty stereotypically high-minded: global politics, literary pursuits, and environmental issues. Also
conspiracy theories, for some reason. That one's weird.

Alright! On to the dumb stuff!

```
> SELECT subreddit, posts, `avg ARI` FROM reddit WHERE posts > 100000 ORDER BY `avg ARI` LIMIT 10;
+-----------------------+----------+---------------------+
|       subreddit       |  posts   |       avg ARI       |
+-----------------------+----------+---------------------+
| me_irl                | 132477   | -6.597215524826628  |
| itookapicture         | 226074   | 2.591935687626967   |
| Fireteams             | 1600719  | 3.1183569978499373  |
| amiugly               | 108643   | 3.135287171164027   |
| Kikpals               | 170943   | 3.3521135496701313  |
| offmychest            | 242639   | 3.794396824263522   |
| Jokes                 | 163818   | 4.192728884240217   |
| 4chan                 | 107196   | 4.337776062292176   |
| GlobalOffensiveTrade  | 823488   | 4.346750167026149   |
| reactiongifs          | 279156   | 4.464272224403798   |
+-----------------------+----------+---------------------+
```

Yup. I can see why these subreddits are dumb. They might not be *bad*, but they're definitely not exactly intellectual. Instead it looks like they focus on funny stuff (jokes, Internet memes, gifs) and personal vanity.

I'd like to close on a more technical note that may help those of you looking to perform similar analyses: If you want
to use a Drill UDF in a cluster setting, as I did here (a six-node HDFS configuration, for those curious) be sure to
copy the relevant .jar files to the `jars/3rdparty` directory of each machine's Drill install. That's all the setup required
to start using the custom functions you've written on big data right away!
