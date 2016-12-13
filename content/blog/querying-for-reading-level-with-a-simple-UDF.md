+++
categories = ["drill", "sql", "udfs", "text"]
date = "2016-02-05"
title = "Querying for reading level with a simple UDF"
type = "blog"
author = "Nathan Griffith"
+++

<script type="text/javascript"
 src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
</script>

Today, just like in [this post](http://www.dremio.com/blog/writing-a-custom-sql-function-for-sentiment-analysis/) from
the previous week, I'd like to discuss creating a simple custom SQL function for Drill that maps strings to float
values. Except this week's function is even *more* simple because it can fit within a single file and requires no
instructions in the `setup()` method. In fact, this may be the simplest example of a Drill UDF I've ever seen, so if
you've been struggling with how to go about writing your own, the source code I'm presenting today maybe a good way to
get some traction.

The raison d'&ecirc;tre of today 's function is to calculate the reading level (or, 'readability') of a single sentence.
Many solutions to the problem of readability utilize syllable counts, which are notoriously difficult to arrive at
computationally. It's possible that a lookup table for those counts would provide satisfactorily speedy results, but the
algorithm that I've chosen to implement, called the automated readability index or ARI, avoids this problem by
altogether using a character count instead. As per the [Wikipedia
article](https://en.wikipedia.org/wiki/Automated_readability_index), the ARI is arrived at via:

$$ ARI = 4.71 \frac{characters}{words} + 0.5 \frac{words}{sentences} - 21.43 $$

However, as I indicated earlier I'm only interested in the readability of single sentences in this particular
application (check out the next article!), so I'm going to implicitly set the number of sentences to 1 in the source code
that comes later.

But before I talk about source you should probably first get some UDF-creation boilerplate out of the way.  I've
discussed how to do this a couple times, but if you're still unsure of what to do go ahead and follow the instructions
in the "Downloading Maven and starting a new project" section near the beginning of [this
article](http://www.dremio.com/blog/writing-a-custom-sql-function-for-sentiment-analysis/).

Once that's out of the way, place this single file (`Readability.java`) in your project's
`main/java/com/yourgroupidentifier/udf` directory:

```java
package com.yourgroupidentifier.udf;

import org.apache.drill.exec.expr.DrillSimpleFunc;
import org.apache.drill.exec.expr.holders.NullableFloat8Holder;
import org.apache.drill.exec.expr.holders.NullableVarCharHolder;

import org.apache.drill.exec.expr.annotations.FunctionTemplate;
import org.apache.drill.exec.expr.annotations.Output;
import org.apache.drill.exec.expr.annotations.Param;

@FunctionTemplate(
        name = "readability",
        scope = FunctionTemplate.FunctionScope.SIMPLE,
        nulls = FunctionTemplate.NullHandling.NULL_IF_NULL
)

public class Readability implements DrillSimpleFunc {

    @Param
    NullableVarCharHolder input;

    @Output
    NullableFloat8Holder out;

    public void setup() {
    }

    public void eval() {

        // The length of 'pneumonoultramicroscopicsilicovolcanoconiosis'
        final int longestWord = 45;

        // Initialize output value
        out.value = 0.0;

        // Split input string up into words
        String inputString = org.apache.drill.exec.expr.fn.impl.StringFunctionHelpers.toStringFromUTF8(input.start,
input.end, input.buffer);
        String[] inputStringWords = inputString.split("\\s+");

        float numWords = inputStringWords.length;
        float numCharacters = inputString.length() - (numWords-1); // Accounts for spaces

        // Adjust for things in the text that aren't words
        // i.e., They are longer than 'longestWord'
        for(int i = 0; i < inputStringWords.length; i++) {
            if( inputStringWords[i].length() > longestWord) {
                numWords--;
                numCharacters = numCharacters - inputStringWords[i].length();
            }
        }

        // Output 'NULL' if the number of words is zero
        if(numWords != 0) {
            out.value = 4.71 * (numCharacters / numWords) + 0.5 * (numWords) - 21.43;
        }
        else {
            out.isSet = 0;
        }
    }
}
```

This is pretty straight forward compared to the [other
examples](http://www.dremio.com/blog/writing-a-custom-sql-function-for-sentiment-analysis/) I've [discussed
before](http://www.dremio.com/blog/querying-google-analytics-json-with-a-custom-sql-function/), right? Just about the
only 'trick' here is that I've made the number the function returns a 'Nullable' type. This is to insure that it has a
more sane output than 'Infinity' when it encounters a field with zero words&mdash;especially useful for when the
function is used in conjunction with `AVG()`, which disregards NULL values but would propagate any 'Infinity' to the
final result.

In the next post, we'll try this function out in the 'field' on one of my favorite data sets!

(And I *swear* I didn't make that pun intentionally.)
