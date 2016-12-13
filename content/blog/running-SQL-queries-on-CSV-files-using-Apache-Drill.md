+++
type = "blog"
author = "Nathan Griffith"
categories = ["sql", "drill", "csv"]
date = "2015-10-13"
title = "Running SQL Queries on CSV Files Using Apache Drill"

+++

If you find a large data store on the web that you're interested in, chances are good that it'll be available for
download as a dump to a CSV file. The CSV format definitely isn't the most sophisticated data storage method around, but
it's common, relatively intuitive for humans to read, and simple to import and export from spreadsheet software like
Excel and LibreOffice. Many of us, however, might want to quickly integrate a CSV data set with an existing database
system (like Hadoop or MongoDB) that's already in place. Or, you may simply prefer to have an SQL interface to the data
instead of suffering through an officeware GUI. In both these cases, Apache Drill is the answer.

For this article I'll focus on the second use scenario as implemented on a single machine running OS X. The data I'll be
looking at comes from a CSV dump of the confirmed planets from NASA's Exoplanet Archive, which is freely available via
[this site](http://exoplanetarchive.ipac.caltech.edu).

To get started, we'll need to install Apache Drill. First, go download and install the Java SE SD Development Kit if you
don't already have it. Then open a terminal and download Drill using the following command

```
$ curl -o apache-drill.tar.gz http://getdrill.org/drill/download/apache-drill-1.1.0.tar.gz
```

Next make an install folder (in this case we're placing it in the home directory)

```
$ mkdir ~/apache-drill
```

and then unpack the file and move the resulting directory's contents to the install location

```
$ tar xzvf apache-drill-1.1.0.tar.gz
$ mv apache-drill-1.1.0/* ~/apache-drill
$ rm -rf apache-drill-1.1.0
```

Now it's easy to launch the single-machine version of Apache Drill with

```
$ ~/apache-drill/bin/drill-embedded
```

From here on out we'll be playing with the file 'planets.csv' that we downloaded from the NASA site. Before the real
number-crunching fun begins, however, you'll need to comment out line 382 of the file (assuming you dumped all of the
columns available). This just hides the names of each field, which would only get in the way since Apache Drill
currently deals directly with the column numbers of a CSV file (this may change in a future release).

Let's test things out with this simple query that shows the number of planets in 10 star systems from the data set:

```
> SELECT columns[1] AS `system name`, columns[4] AS `number of planets` FROM dfs.`/Users/nategri/dremio/playdata/planets.csv` LIMIT 20;
+----------------------------+--------------------+
|        system name         | number of planets  |
+----------------------------+--------------------+
| 11 Com                     | 1                  |
| 11 UMi                     | 1                  |
| 14 And                     | 1                  |
| 14 Her                     | 1                  |
| 16 Cyg B                   | 1                  |
| 18 Del                     | 1                  |
| 1RXS J160929.1-210524      | 1                  |
| 24 Sex                     | 2                  |
| 24 Sex                     | 2                  |
| 2MASS J01225093-2439505    | 1                  |
| 2MASS J02192210-3925225    | 1                  |
| 2MASS J04414489+2301513    | 1                  |
| 2MASS J12073346-3932539    | 1                  |
| 2MASS J19383260+4603591    | 1                  |
| 2MASS J21402931+1625183 A  | 1                  |
| 30 Ari B                   | 1                  |
| 4 UMa                      | 1                  |
| 42 Dra                     | 1                  |
| 47 UMa                     | 3                  |
| 47 UMa                     | 3                  |
+----------------------------+--------------------+
20 rows selected (0.141 seconds)
```

Our solar system has 8 planets, but it seems like a lot of these only have 1 or 2. Let's run another query to get
the average number of planets (note: I'm using CAST here to convert the strings to floats).

```
> SELECT AVG(CAST(columns[4] AS float)) AS `avg num of planets` FROM dfs.`/Users/nategri/dremio/playdata/planets.csv`;
+---------------------+
| avg num of planets  |
+---------------------+
| 2.134249471458774   |
+---------------------+
1 row selected (0.15 seconds)

```

Yes, that is certainly far less than 8. So is our solar system a freak? Well, probably not. It's easy to imagine that
bigger planets are simply easier to detect, and if this is the case (spoiler: this definitely the case) then the average
mass of planets in systems with only one detected body ought to be quite a bit bigger than those with more. We can test
our "hypothesis" (that we know to be true) a with a more complex query:

```
> SELECT columns[4] AS `num planets`, AVG(CAST(columns[101] AS float)) AS `avg planet mass`, COUNT(DISTINCT columns[1]) AS `num star systems` FROM dfs.`/Users/nategri/dremio/playdata/planets.csv` WHERE columns[101] <> '' GROUP BY > columns[4] ORDER BY CAST(columns[4] AS float);
+--------------+---------------------+-------------------+
| num planets  |   avg planet mass   | num star systems  |
+--------------+---------------------+-------------------+
| 1            | 954.3903533472983   | 332               |
| 2            | 631.1303849087407   | 50                |
| 3            | 460.048579923199    | 24                |
| 4            | 542.5327862175182   | 11                |
| 5            | 158.82839210055494  | 8                 |
| 6            | 7.8500000437100725  | 1                 |
+--------------+---------------------+-------------------+
6 rows selected (0.801 seconds)
```

Indeed, systems with one detected planet dominate the data set, and the average mass of their planets (in units of
Jupiter masses) is *far* higher than that of systems with more planets. Apache Drill is amazing for exactly this kind of
work: spotting and verifying features in found data sets in a rapid, interactive fashion.

Anyway that's it for this tutorial. Happy Drilling!
