+++
title = "Calculating Pearson's r using a custom SQL function"
type = "blog"
author = "Nathan Griffith"
categories = ["drill", "sql", "udfs", "custom functions", "pearson's r", "correlation coefficient", "statistics"]
date = "2016-02-10"
+++

<script type="text/javascript"
 src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
</script>

Lately I've written a lot of custom functions to assist me in my example Drill analyses, but they've all been of the
same fundamental type: They take one or more columns of a single row and process them into a single output. The [Drill
documentation](https://drill.apache.org/docs/develop-custom-functions-introduction/) calls these "simple" functions.
However there's another class of functions lurking out there&mdash;ones that can accept *many* rows of data as input. We
call them "aggregate" functions.

If you're an experienced user of SQL, you're already familiar with a few very common aggregate functions like `COUNT()`
and `SUM()`, but you've probably never written one of your own. Today we're going to change that!

As I've discussed in previous articles Drill already has some built-in statistics functions, but the goal of this post
will be to expand those capabilities even further by implementing an aggregate function to calculate a value called
Pearson's *r*. Values for *r* vary from +1 to -1, and indicate the degree to which two variables are linearly correlated
or anti-correlated, respectively. An *r* value at or near 0 indicates that there is no linear relationship between the
two sets of data points.

After looking on Wikipedia, the most Drill-friendly equation for Pearson's *r* is:

$$ r = \frac{n \sum x_i y_i - \sum x_i \sum y_i}{ \sqrt{n \sum x_i^2 - \left( \sum x_i \right)^2} \sqrt{n \sum y_i^2 -
\left( \sum y_i \right)^2}} $$

where \\( x_i \\) and \\( y_i \\) are our data points, and \\( n \\) is the total number of them.

Once you've got a Maven project started for your Drill UDF (a guide is available in the "Downloading Maven and starting
a new project" section of [this
article](http://www.dremio.com/blog/writing-a-custom-sql-function-for-sentiment-analysis/)), take a look at the source
for our Pearson's *r* function:

```java
package com.yourgroupidentifier.udf;

import org.apache.drill.exec.expr.DrillAggFunc;
import org.apache.drill.exec.expr.holders.IntHolder;
import org.apache.drill.exec.expr.holders.NullableFloat8Holder;
import org.apache.drill.exec.expr.holders.Float8Holder;

import org.apache.drill.exec.expr.annotations.FunctionTemplate;

import org.apache.drill.exec.expr.annotations.Param;
import org.apache.drill.exec.expr.annotations.Workspace;
import org.apache.drill.exec.expr.annotations.Output;

@FunctionTemplate(
        name = "pcorrelation",
        scope = FunctionTemplate.FunctionScope.POINT_AGGREGATE,
        nulls = FunctionTemplate.NullHandling.INTERNAL
)

public class PCorrelation implements DrillAggFunc {

    @Param
    NullableFloat8Holder xInput;

    @Param
    NullableFloat8Holder yInput;

    @Workspace
    IntHolder numValues;

    @Workspace
    Float8Holder xSum;

    @Workspace
    Float8Holder ySum;

    @Workspace
    Float8Holder xSqSum;

    @Workspace
    Float8Holder ySqSum;

    @Workspace
    Float8Holder xySum;

    @Output
    Float8Holder output;

    public void setup() {
        // Initialize values
        numValues.value = 0;
        xSum.value = 0;
        ySum.value = 0;
        xSqSum.value = 0;
        ySqSum.value = 0;
        xySum.value = 0;
    }

    public void reset() {
        // Initialize values
        numValues.value = 0;
        xSum.value = 0;
        ySum.value = 0;
        xSqSum.value = 0;
        ySqSum.value = 0;
        xySum.value = 0;
    }

    public void add() {

        // Only proceed if both floats aren't nulls
        if( (xInput.isSet == 1) || (yInput.isSet == 1) ) {

            numValues.value++;

            xSum.value += xInput.value;
            ySum.value += yInput.value;

            xSqSum.value += xInput.value * xInput.value;
            ySqSum.value += yInput.value * yInput.value;

            xySum.value += xInput.value * yInput.value;
        }

    }

    public void output() {

        float n = numValues.value;

        double x = xSum.value;
        double y = ySum.value;

        double x2 = xSqSum.value;
        double y2 = ySqSum.value;

        double xy = xySum.value;

        output.value = (n*xy - x*y)/(Math.sqrt(n*x2 - x*x)*Math.sqrt(n*y2 - y*y));
    }
}
```

Yes, that's a chunk of code&mdash;but it's mostly this long because it takes a lot of variables to accomplish the *r*
calculation. Anyway, let's talk about some differences between this aggregate function and the simple ones we've been
writing up until now.

First, up in the function template the scope changes from `SIMPLE` to `POINT_AGGREGATE`, while the null handling is set
to `INTERNAL` instead of `NULL_IF_NULL`. This is because aggregate functions need to determine on their own how to
process null inputs, rather than let Drill handle it for them as we can do for most simple functions. You'll also notice
a new annotation, `@Workspace`, which is used before variables that assist in the calculation of the result as the
function moves through each row.

Another obvious difference is that aggregate functions implement a different set of methods than simple ones. The
`setup()` method remains the same, but `output()` takes the place of `eval()`. For each row that's processed `add()` is
called, and `reset()` is used to determine what the function does when it hits a new set of rows.

In the next article, I'll take this new `PCORRELATION()` function out for a spin on some vehicle data from the EPA.

(OK, yes, pun very much intended that time.)
