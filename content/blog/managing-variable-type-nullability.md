+++
type = "blog"
author = "Nathan Griffith"
categories = ["drill", "sql", "udfs", "custom functions", "types", "variables", "null", "nullable"]
date = "2016-02-09"
title = "Managing variable type nullability"
+++

One day you may find yourself with a custom function for Drill that's very particular about the kind of variables that
it accepts. In particular, it may hold strong opinions about whether or not a variable is allowed to express a `NULL`
value. In fact it may even be *you* who wrote this unavoidably fussy function (SPOILER: This is exactly what happened to
me earlier this week).

Currently Drill lacks built-in functions to add or strip nullability from variables, but luckily it's very easy to whip
up a couple UDFs which do exactly that. Today I'll be showcasing two such functions which respectively add and remove
nullability from Drill `FLOAT` variables. As usual you should perform the necessary incantations and summoning rituals
for creating a custom function project in Maven (see the section "Downloading Maven and starting a new project" in [this
article](http://www.dremio.com/blog/writing-a-custom-sql-function-for-sentiment-analysis/) for a refresher).

Once you're ready to start thinking about source code, the class associated with the function to add nullability looks
like:

```java
package com.yourgroupidentifier.udf;

import org.apache.drill.exec.expr.DrillSimpleFunc;
import org.apache.drill.exec.expr.holders.NullableFloat8Holder;
import org.apache.drill.exec.expr.holders.Float8Holder;

import org.apache.drill.exec.expr.annotations.FunctionTemplate;
import org.apache.drill.exec.expr.annotations.Output;
import org.apache.drill.exec.expr.annotations.Param;

@FunctionTemplate(
        name = "add_null_float",
        scope = FunctionTemplate.FunctionScope.SIMPLE,
        nulls = FunctionTemplate.NullHandling.INTERNAL
)

public class addNullFloat implements DrillSimpleFunc {

    @Param
    Float8Holder input;

    @Output
    NullableFloat8Holder output;

    public void setup() {
    }

    public void eval() {
        output.isSet = 1;
        output.value = input.value;
    }
}
```

while the one for removing nullability is as follows:

```java
package com.yourgroupidentifier.udf;

import org.apache.drill.exec.expr.DrillSimpleFunc;
import org.apache.drill.exec.expr.holders.NullableFloat8Holder;
import org.apache.drill.exec.expr.holders.Float8Holder;

import org.apache.drill.exec.expr.annotations.FunctionTemplate;
import org.apache.drill.exec.expr.annotations.Output;
import org.apache.drill.exec.expr.annotations.Param;

@FunctionTemplate(
        name = "remove_null_float",
        scope = FunctionTemplate.FunctionScope.SIMPLE,
        nulls = FunctionTemplate.NullHandling.INTERNAL
)

public class removeNullFloat implements DrillSimpleFunc {

    @Param
    NullableFloat8Holder input;

    @Output
    Float8Holder output;

    public void setup() {
    }

    public void eval() {
        output.value = input.value;
    }
}
```

Remember back in [this post](http://www.dremio.com/blog/writing-a-custom-sql-function-for-sentiment-analysis/) when I
thought I was presenting the simplest UDFs I'd ever seen demonstrated? Well, these guys are definitely setting a new
world record. They're just about the smallest custom functions you can code for Drill. But that doesn't mean they're not
useful! `ADD_NULL_FLOAT()` and `REMOVE_NULL_FLOAT()` make valuable additions to a Drill power user's toolbox, and I'll
be putting one of them to work in a post later this week.
