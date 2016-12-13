+++
author = "Nathan Griffith"
categories = ["drill", "sql", "udfs", "custom functions", "pearson's r", "correlation coefficient", "statistics", "cars", "epa", "mpg"]
date = "2016-02-11"
title = "Parsing EPA vehicle data for linear correlations in MPG ratings"
type = "blog"
+++

In [the previous day's post](http://www.dremio.com/blog/calculating-pearsons-r-using-a-custom-sql-function/) I
demonstrated how to code a custom aggregate function that can process two sets of data points into their corresponding
Pearson's *r* value, which is a useful indicator of variable correlation. Today I'm going to put that function to the
test on this [EPA data set](https://www.fueleconomy.gov/feg/download.shtml) that contains information about vehicles
manufactured from model years 1984 to 2017. The two questions I'd like to answer are: 1.) Does the combined MPG rating
for a car correlate with model year? and 2.) How does an engine's displacement affect the combined MPG rating?

To begin with, I'm going to create two views that average over combined MPG for each of the other variables of interest.
These are constructed as follows:

```
CREATE VIEW year_mpg AS
     SELECT CAST(`year` AS FLOAT) `year`, AVG(CAST(comb08 AS FLOAT)) mpg
       FROM dfs.`/Users/ngriffith/Downloads/vehicles.csvh`
   GROUP BY `year`
   ORDER BY `year`;
```

helps answer the first question about model years, while

```
CREATE VIEW displacement_mpg AS
     SELECT CAST(displ AS FLOAT) displacement, AVG(CAST(comb08 AS FLOAT)) mpg
       FROM dfs.`/Users/ngriffith/Downloads/vehicles.csvh`
      WHERE displ NOT LIKE ''
        AND displ NOT LIKE 'NA'
   GROUP BY displ
   ORDER BY displacement;
```

will allow us tackle the second one about engine sizes.

The first view looks like this:

```
> SELECT * FROM year_mpg;
+---------+---------------------+
|  year   |         mpg         |
+---------+---------------------+
| 1984.0  | 19.881873727087576  |
| 1985.0  | 19.808348030570254  |
| 1986.0  | 19.550413223140495  |
| 1987.0  | 19.228548516439453  |
| 1988.0  | 19.328318584070797  |
| 1989.0  | 19.12575888985256   |
| 1990.0  | 19.000927643784788  |
| 1991.0  | 18.825971731448764  |
| 1992.0  | 18.86262265834077   |
| 1993.0  | 19.104300091491307  |
| 1994.0  | 19.0122199592668    |
| 1995.0  | 18.797311271975182  |
| 1996.0  | 19.584734799482536  |
| 1997.0  | 19.429133858267715  |
| 1998.0  | 19.51847290640394   |
| 1999.0  | 19.61150234741784   |
| 2000.0  | 19.526190476190475  |
| 2001.0  | 19.479692645444565  |
| 2002.0  | 19.168205128205127  |
| 2003.0  | 19.00095785440613   |
| 2004.0  | 19.067736185383243  |
| 2005.0  | 19.193825042881645  |
| 2006.0  | 18.95923913043478   |
| 2007.0  | 18.97868561278863   |
| 2008.0  | 19.27632687447346   |
| 2009.0  | 19.74070945945946   |
| 2010.0  | 20.601442741208295  |
| 2011.0  | 21.10353982300885   |
| 2012.0  | 21.93755420641804   |
| 2013.0  | 23.253164556962027  |
| 2014.0  | 23.70114006514658   |
| 2015.0  | 24.214953271028037  |
| 2016.0  | 24.84784446322908   |
| 2017.0  | 23.571428571428573  |
+---------+---------------------+
```

Yup, it definitely looks like there's a trend toward higher MPG. But let's calculate the *r* value:

```
> SELECT PCORRELATION(`year`, mpg) FROM year_mpg;
Error: SYSTEM ERROR: SchemaChangeException: Failure while materializing expression.
Error in expression at index -1.  Error: Missing function implementation: [pcorrelation(FLOAT4-REQUIRED,
FLOAT8-OPTIONAL)].  Full expression: --UNKNOWN EXPRESSION--.
...
```

Oops! We got an error message instead of an *r* value. That's because my `PCORRELATION()` function expects two variables
that are nullable ('optional'), but the first one that's getting passed is non-nullable ('required'). This situation is
what led to the creation of [this earlier article and its associated
functions](http://www.dremio.com/blog/managing-variable-type-nullability/) for stripping and adding nullability to
variables. The `ADD_NULL_FLOAT()` custom function from that piece is exactly what we need to turn `` `years ` `` into a
variable type that `PCORRELATION()` can accept.

So let's try this again:

```
> SELECT PCORRELATION(ADD_NULL_FLOAT(`year`), mpg) FROM year_mpg;
+---------------------+
|       EXPR$0        |
+---------------------+
| 0.6870535033886027  |
+---------------------+
```

Correlation confirmed! It's not the strongest (that would be a value of 1.0), but it's definitely there. Neat!

Now to take a look at how engine size related. The view I created contains this data:

```
> SELECT * FROM displacement_mpg;
+---------------+---------------------+
| displacement  |         mpg         |
+---------------+---------------------+
| 0.0           | 87.5                |
| 0.6           | 39.0                |
| 0.9           | 35.5                |
| 1.0           | 37.5030303030303    |
| 1.1           | 19.916666666666668  |
| 1.2           | 30.81081081081081   |
| 1.3           | 28.904255319148938  |
| 1.4           | 30.658959537572255  |
| 1.5           | 29.439592430858806  |
| 1.6           | 26.48841961852861   |
| 1.7           | 28.842105263157894  |
| 1.8           | 24.91231463571889   |
| 1.9           | 27.038277511961724  |
| 2.0           | 24.032035928143713  |
| 2.1           | 19.28301886792453   |
| 2.2           | 22.07820419985518   |
| 2.3           | 21.026008968609865  |
| 2.4           | 22.373468300479487  |
| 2.5           | 21.628227194492254  |
| 2.6           | 18.123529411764707  |
| 2.7           | 19.88589211618257   |
| 2.8           | 18.463019250253293  |
| 2.9           | 18.772727272727273  |
| 3.0           | 19.428571428571427  |
| 3.1           | 19.72156862745098   |
| 3.2           | 18.25237191650854   |
| 3.3           | 18.54698795180723   |
| 3.4           | 18.473317865429234  |
| 3.5           | 20.02212705210564   |
| 3.6           | 19.188457008244995  |
| 3.7           | 18.107468123861565  |
| 3.8           | 19.115025906735752  |
| 3.9           | 15.588815789473685  |
| 4.0           | 16.738241308793455  |
| 4.1           | 15.873684210526315  |
| 4.2           | 16.05597014925373   |
| 4.3           | 16.590775988286968  |
| 4.4           | 17.094736842105263  |
| 4.5           | 15.566666666666666  |
| 4.6           | 16.6696269982238    |
| 4.7           | 15.282548476454293  |
| 4.8           | 15.813688212927756  |
| 4.9           | 14.355113636363637  |
| 5.0           | 15.2375             |
| 5.2           | 13.063197026022305  |
| 5.3           | 15.203412073490814  |
| 5.4           | 13.902597402597403  |
| 5.5           | 15.16243654822335   |
| 5.6           | 14.0                |
| 5.6           | 14.394736842105264  |
| 5.7           | 14.780718336483933  |
| 5.8           | 11.78125            |
| 5.9           | 11.701183431952662  |
| 6.0           | 14.462686567164178  |
| 6.1           | 15.0                |
| 6.1           | 14.454545454545455  |
| 6.2           | 16.540930979133226  |
| 6.3           | 13.705882352941176  |
| 6.4           | 16.8                |
| 6.5           | 14.81081081081081   |
| 6.6           | 15.0                |
| 6.7           | 13.0                |
| 6.8           | 10.572463768115941  |
| 7.0           | 17.4                |
| 7.4           | 9.75                |
| 8.0           | 12.347826086956522  |
| 8.3           | 11.222222222222221  |
| 8.4           | 15.6                |
+---------------+---------------------+
```

Which, after making a similar adjustment using `ADD_NULL_FLOAT()`, yields a correlation of:

```
> SELECT PCORRELATION(ADD_NULL_FLOAT(displacement), mpg) FROM displacement_mpg;
+---------------------+
|       EXPR$0        |
+---------------------+
| -0.679501632464044  |
+---------------------+
```

So combined average combined MPG is almost as strongly anti-correlated with engine size as it is correlated with model
year. There's a fairly clear linear dependence in both cases!

For my next couple articles I'll be digging even deeper into statistical techniques by implementing and testing a Drill
function to calculate the probability of events that follow a Poisson distribution. It's shaping up to be a great
couple of weeks to be reading the Dremio Blog if you're curious about what custom-tuned 'SQL on anything' software can
do in terms of serious analysis.
