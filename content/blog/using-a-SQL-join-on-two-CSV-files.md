+++
author = "Nathan Griffith"
categories = ["drill", "sql", "csv", "join", "files"]
date = "2016-01-19"
title = "Using a SQL JOIN on two CSV files"
type = "blog"
+++

One of Drill's great strengths is its ability to unite sets of information found in different places. In today's article
I'll provide a quick example of this functionality by showing you how to you use Drill to quickly improve the interface
to some found data.

We'll start with the interesting but strangely formatted business-centered census data that we used in the [last
post](http://www.dremio.com/blog/converting-csv-file-data-to-kudu-storage/) (available on
[this site](https://www.census.gov/econ/cbp/download/) as a 14.3 MB zip file). As you'll recall, our command to query
the state, county, and industry codes, along with the number of establishments, yielded results in the following format:

```
+-----------+----------+---------+------+
| fipstate  | fipscty  |  naics  | est  |
+-----------+----------+---------+------+
| 55        | 107      | 321113  | 1    |
| 48        | 185      | 51213/  | 1    |
| 39        | 135      | 42491/  | 6    |
| 04        | 027      | 4411//  | 30   |
| 08        | 085      | 541870  | 1    |
| 26        | 061      | 51----  | 17   |
| 28        | 049      | 519///  | 3    |
| 48        | 139      | 48849/  | 3    |
| 22        | 073      | 4853//  | 3    |
| 48        | 139      | 424910  | 10   |
+-----------+----------+---------+------+
```

It's great to have data with this level of granularity, but the FIPS codes that the file uses to identify the state and
county are far from intuitive. Thankfully we can use Drill in combination with this [lovely CSV
file](https://github.com/hadley/data-counties/blob/master/county-fips.csv) from Github user 'hadley' to make the data
more user-friendly. (Remember to rename both files as '.csvh' so Drill knows to pick up the column names!)

Because Drill effectively treats each file as a separate table, we can gain access to human readable state and county
names by performing an SQL JOIN across the files.

That handy file from Github has a format like

```
+-----------------------+--------+--------------+-------------+
|        county         | state  | county_fips  | state_fips  |
+-----------------------+--------+--------------+-------------+
| Aleutians East        | AK     | 13           | 2           |
| Aleutians West        | AK     | 16           | 2           |
| Anchorage             | AK     | 20           | 2           |
| Bethel                | AK     | 50           | 2           |
| Bristol Bay           | AK     | 60           | 2           |
| Denali                | AK     | 68           | 2           |
| Dillingham            | AK     | 70           | 2           |
| Fairbanks North Star  | AK     | 90           | 2           |
| Haines                | AK     | 100          | 2           |
| Juneau                | AK     | 110          | 2           |
+-----------------------+--------+--------------+-------------+
```

so a simple query that brings state and county names to the census data using a JOIN is:

```
  SELECT t2.state, t2.county, SUM(CAST(t1.est AS INT)) `total establishments`
    FROM dfs.`/path/to/cbp13co.csvh` t1
    JOIN dfs.`/path/to/county-fips.csvh` t2
      ON t1.fipstate = t2.state_fips
     AND t1.fipscty = t2.county_fips
GROUP BY t2.state, t2.county
ORDER BY `total establishments`
    DESC LIMIT 10;
```

Which, for the curious, produces the following output:

```
+--------+--------------+-----------------------+
| state  |    county    | total establishments  |
+--------+--------------+-----------------------+
| TX     | Harris       | 571756                |
| TX     | Dallas       | 374670                |
| NY     | Suffolk      | 291970                |
| TX     | Tarrant      | 232484                |
| MI     | Oakland      | 230154                |
| GA     | Fulton       | 203028                |
| MI     | Wayne        | 190806                |
| NY     | Westchester  | 189820                |
| TX     | Travis       | 186236                |
| MO     | St. Louis    | 184326                |
+--------+--------------+-----------------------+
```

Cool! (And, no, I have no idea why Harris county, Texas is apparently the business capitol of the United States.)
