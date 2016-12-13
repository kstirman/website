+++
type = "blog"
author = "Nathan Griffith"
categories = ["drill", "sql", "date", "time", "format", "crime", "san francisco"]
date = "2016-02-02"
title = "The case of the stolen candy hearts: Advanced date parsing in SQL"
+++

The other day I had 12 years of [San Francisco crime
data](https://data.sfgov.org/Public-Safety/Map-Crime-Incidents-from-1-Jan-2003/gxxq-x39z) loaded in Drill and I wanted
to answer the following question: Which days from recent years have the highest incidences of crime?

As it turns out, this isn't that difficult to accomplish, but it did add some new functions to my repertoire, so I
thought I'd share the process with you.

Once I got a hold of the SF crime download, I renamed it to a file with a '.csvh' extension so I could address the data
by the column name given in the header. And as we can see in this simple query of the data

```
> SELECT Category, `Date`, `Time`, Address FROM dfs.`/path/to/sfcrime.csvh` LIMIT 5;
+------------------------------+-------------+--------+---------------------------+
|           Category           |    Date     |  Time  |          Address          |
+------------------------------+-------------+--------+---------------------------+
| VANDALISM                    | 01/14/2016  | 23:45  | 3600 Block of ALEMANY BL  |
| ASSAULT                      | 01/14/2016  | 23:45  | 0 Block of DRUMM ST       |
| OTHER OFFENSES               | 01/14/2016  | 23:29  | PALOU AV / LANE ST        |
| DRIVING UNDER THE INFLUENCE  | 01/14/2016  | 23:29  | PALOU AV / LANE ST        |
| OTHER OFFENSES               | 01/14/2016  | 23:00  | 100 Block of DAKOTA ST    |
+------------------------------+-------------+--------+---------------------------+
```

the column labeled 'Date' follows the 'MM/dd/yyy' format, so I'll want to keep that in mind when I use the `TO_DATE()`
function to transform that entry from a string to a `DATE`.

But remember, the ultimate goal is to find out which days of a given year have the highest crime. To do this I'll need
to make use of the `EXTRACT()` function to pull the month, day, and year number from my newly constructed `DATE` types.
This is the function that was new to me, but thankfully it's very easy to understand. You just specify which component
of the date you'd like to pull as part of the argument, as in:

```
EXTRACT(day FROM myDate)
```

I ended up converting the 'Date' column and performing the necessary EXTRACTs at the same time in this view (remember to
`USE dfs.tmp;` before entering this command):

```
CREATE VIEW crimedays AS
     SELECT EXTRACT(month FROM TO_DATE(`Date`,'MM/dd/yyyy')) month_num, EXTRACT(day FROM TO_DATE(`Date`,'MM/dd/yyyy')) day_num,
            EXTRACT(year FROM TO_DATE(`Date`,'MM/dd/yyyy')) year_num, IncidntNum id, Category type
       FROM dfs.`/path/to/sfcrime.csvh`;
```

So now all it takes is a query with two GROUP BYs on the month and day number to come up with a list of high crime days
for a previous year. Let's take a look at 2014:

```
  SELECT month_num, day_num, year_num, COUNT(id) crimes
    FROM crimedays
   WHERE year_num = 2014
GROUP BY month_num, day_num, year_num
ORDER BY crimes DESC
   LIMIT 5;
+------------+----------+-----------+---------+
| month_num  | day_num  | year_num  | crimes  |
+------------+----------+-----------+---------+
| 10         | 11       | 2014      | 521     |
| 2          | 14       | 2014      | 514     |
| 3          | 19       | 2014      | 513     |
| 8          | 8        | 2014      | 511     |
| 8          | 9        | 2014      | 509     |
+------------+----------+-----------+---------+
```

Apparently February 14th was an especially high crime day that year. So this coming Valentine's Day, don't forget to buy
your significant other something nice. But also maybe take some extra care making sure no one steals it before you can
give it to them!
