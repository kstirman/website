+++
date = "2015-12-01"
title = "Bless this mess: Working with complicated JSON structure in SQL"
author = "Nathan Griffith"
categories = ["json", "sql", "drill", "mongodb", "views"]
type = "blog"

+++

Data you find on the web comes in all formats. From simple CSV to some really creepy-crawly JSON. The data provided by
the Twitter API to describe a tweet unfortunately falls within the latter category.

In this article I'll use Drill to provide an SQL interface to a MongoDB collection of about 52,000 tweets pulled from
the Twitter streaming API. As I alluded to, tweet data presents itself in a way that manifests a lot of nested elements,
and documents in the collection follow this JSON structure:

```
{
  "contributors": null,
  "coordinates": null,
  "created_at": "Mon Nov 09 23:18:16 +0000 2015",
  "entities": {
    "hashtags": [],
    "media": [
      {
        "display_url": "pic.twitter.com/NNAPr46qDz",
        "expanded_url": "http://twitter.com/NewsFlashback/status/663858171639939072/photo/1",
        "id": 663858170704490496,
        "id_str": "663858170704490496",
        "indices": [
          121,
          144
        ],
        "media_url": "http://pbs.twimg.com/media/CTZ_fS4U8AAkti7.jpg",
        "media_url_https": "https://pbs.twimg.com/media/CTZ_fS4U8AAkti7.jpg",
        "sizes": {
          "large": {
            "h": 450,
            "resize": "fit",
            "w": 450
          },
          "medium": {
            "h": 450,
            "resize": "fit",
            "w": 450
          },
          "small": {
            "h": 340,
            "resize": "fit",
            "w": 340
          },
          "thumb": {
            "h": 150,
            "resize": "crop",
            "w": 150
          }
        },
        "type": "photo",
        "url": "https://t.co/NNAPr46qDz"
      }
    ],
    ... etc
```


(see [this link](https://gist.github.com/nategri/c34e1868792b23e97f2f) for the whole entry). That looks pretty intense,
right? But don't worry&mdash;Drill is up to the task! We'll be talking to that mess with ANSI SQL in no time.

Assuming you've already connected Drill to your MongoDB (just click 'Enable' next to the MongoDB plugin at
http://localhost:8047/storage after running `drill-embedded`), we're ready to begin querying the Twitter data using SQL.

For my analysis goal, I’m going focus on coming up with a method to examine the number of hashtags that occur in tweets
from verified users in my data set. To begin, let’s look at the array of hashtags that the tweet JSON lists for verified
users with at least one hashtag:

```sql
SELECT tw.id `Tweet Id`, tw.`user`.screen_name `Screen Name`, tw.entities.hashtags `Hashtag Array`
  FROM dfs.`/path/to/tweets.small.json` tw
 WHERE tw.entities.hashtags[0].text IS NOT NULL
   AND tw.`user`.verified = true
 LIMIT 15;
```

With results:

```
+---------------------+----------------+-----------------------------------------------------------------------------------+
|      Tweet Id       |  Screen Name   |                                   Hashtag Array                                   |
+---------------------+----------------+-----------------------------------------------------------------------------------+
| 663858238757199873  | FinishLine     | [{"indices":[116,132],"text":"TheFundamentals"}]                                  |
| 663858343581237248  | antena3com     | [{"indices":[82,97],"text":"MarDePlástico8"}]                                     |
| 663858351978078208  | tacobell       | [{"indices":[12,28],"text":"TacoEmojiEngine"}]                                    |
| 663858351990697984  | tacobell       | [{"indices":[14,30],"text":"TacoEmojiEngine"}]                                    |
| 663858553329815553  | CollegeHumor   | [{"indices":[0,14],"text":"TheCrunchBowl"}]                                       |
| 663858746251042816  | tacobell       | [{"indices":[12,28],"text":"TacoEmojiEngine"}]                                    |
| 663859039831486465  | VctorClavijo   | [{"indices":[14,23],"text":"Carlos10"}]                                           |
| 663859115358158848  | FarnamHorse    | [{"indices":[0,11],"text":"DidYouKnow"}]                                          |
| 663859161503928320  | vpelham        | [{"indices":[20,29],"text":"feminist"},{"indices":[127,136],"text":"feminism"}]   |
| 663859312469651456  | AlPrimerToque  | [{"indices":[0,18],"text":"LorenzoEnOndaCero"}]                                   |
| 663859425715834880  | NewsTalk770    | [{"indices":[82,91],"text":"yycroads"},{"indices":[92,103],"text":"yyctraffic"}]  |
| 663859471844810752  | BrentASJax     | [{"indices":[70,83],"text":"FirstAlertWX"}]                                       |
| 663859530598514688  | simpsonwhnt    | [{"indices":[102,111],"text":"valleywx"}]                                         |
| 663859702535553024  | PrimerImpacto  | [{"indices":[75,89],"text":"PrimerImpacto"}]                                      |
| 663859799004581888  | DPostSports    | [{"indices":[41,49],"text":"Rockies"}]                                            |
+---------------------+----------------+-----------------------------------------------------------------------------------+
```

Now I’ll use the `FLATTEN()` function on the array to make one row for each hashtag in a tweet:

```sql
SELECT tw.id `Tweet Id`, tw.`user`.screen_name `Screen Name`, FLATTEN(tw.entities.hashtags) `Hashtags`
  FROM dfs.`/path/to/tweets.small.json` tw
 WHERE tw.entities.hashtags[0].text IS NOT NULL
   AND tw.`user`.verified = true
 LIMIT 15;
```

```
+---------------------+----------------+-------------------------------------------------+
|      Tweet Id       |  Screen Name   |                    Hashtags                     |
+---------------------+----------------+-------------------------------------------------+
| 663858238757199873  | FinishLine     | {"indices":[116,132],"text":"TheFundamentals"}  |
| 663858343581237248  | antena3com     | {"indices":[82,97],"text":"MarDePlástico8"}     |
| 663858351978078208  | tacobell       | {"indices":[12,28],"text":"TacoEmojiEngine"}    |
| 663858351990697984  | tacobell       | {"indices":[14,30],"text":"TacoEmojiEngine"}    |
| 663858553329815553  | CollegeHumor   | {"indices":[0,14],"text":"TheCrunchBowl"}       |
| 663858746251042816  | tacobell       | {"indices":[12,28],"text":"TacoEmojiEngine"}    |
| 663859039831486465  | VctorClavijo   | {"indices":[14,23],"text":"Carlos10"}           |
| 663859115358158848  | FarnamHorse    | {"indices":[0,11],"text":"DidYouKnow"}          |
| 663859161503928320  | vpelham        | {"indices":[20,29],"text":"feminist"}           |
| 663859161503928320  | vpelham        | {"indices":[127,136],"text":"feminism"}         |
| 663859312469651456  | AlPrimerToque  | {"indices":[0,18],"text":"LorenzoEnOndaCero"}   |
| 663859425715834880  | NewsTalk770    | {"indices":[82,91],"text":"yycroads"}           |
| 663859425715834880  | NewsTalk770    | {"indices":[92,103],"text":"yyctraffic"}        |
| 663859471844810752  | BrentASJax     | {"indices":[70,83],"text":"FirstAlertWX"}       |
| 663859530598514688  | simpsonwhnt    | {"indices":[102,111],"text":"valleywx"}         |
+---------------------+----------------+-------------------------------------------------+
```

You can probably guess where I’m heading by now. A `COUNT()` and `GROUP BY` on a table like this can give us the number
hashtags in each tweet. So let’s make a temporary reference to this query’s output with Drill’s 'view' functionality. To
do this we switch to the local file system’s temporary workspace and then run a CREATE VIEW command:

```sql
> USE dfs.tmp;
> CREATE VIEW hashtags AS SELECT tw.id `Tweet Id`, tw.`user`.screen_name `Screen Name`, FLATTEN(tw.entities.hashtags) `Hashtags` FROM dfs.`/path/to/tweets.small.json` tw WHERE tw.entities.hashtags[0].text IS NOT NULL AND tw.`user`.verified = true;
```

And now we’re ready to query our 'hashtags' view to sort verified tweets by the number of hashtags.

```sql
  SELECT `Tweet Id`, `Screen Name`, COUNT(`Tweet Id`) `Num Hashtags`
    FROM hashtags
GROUP BY `Tweet Id`, `Screen Name`
ORDER BY `Num Hashtags` DESC
   LIMIT 15;
```

```
+---------------------+------------------+---------------+
|      Tweet Id       |   Screen Name    | Num Hashtags  |
+---------------------+------------------+---------------+
| 663860369459384320  | SOFIALAMA        | 4             |
| 663861480945664004  | ChrisMillerKUTV  | 3             |
| 663861938103930880  | XboxFR           | 3             |
| 663862705674133504  | sputnik_TR       | 2             |
| 663862542079557633  | WHO              | 2             |
| 663862743427076096  | ljcisneros       | 2             |
| 663861338310053888  | joelcomm         | 2             |
| 663859161503928320  | vpelham          | 2             |
| 663861027948249088  | joemaalouftv     | 2             |
| 663859425715834880  | NewsTalk770      | 2             |
| 663861791303274497  | DezMandamentos   | 2             |
| 663860684006952960  | RogerCookMLA     | 2             |
| 663859799004581888  | DPostSports      | 1             |
| 663859702535553024  | PrimerImpacto    | 1             |
| 663859039831486465  | VctorClavijo     | 1             |
+---------------------+------------------+---------------+
```

Ta da!

But views aren’t just for simplifying SQL statements. For instance, you could also employ a view to create a shortcut to
a query that joins several different files and/or databases. And since views only store queries and not data, you can
make a lot of them without worrying about consuming the amount of disk space associated with table creation.

So to recap: In this post I've showcased how to deal with complicated nested JSON data in Drill, while also
demonstrating how a 'view' can improve the flow and readability of your analysis.
