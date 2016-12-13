+++
author = "Nathan Griffith"
categories = ["drill", "google analytics", "sql", "java"]
date = "2015-10-30"
title = "Querying Google Analytics JSON with a custom SQL function"
type = "blog"

+++

[Last time](/blog/using-sql-to-interface-with-google-analytics-data-stored-on-amazon-s3/) I wrapped up
by showing you how to look at Google Analytics JSON using a nested SQL query in Apache Drill.  This approach is fine,
but by implementing a custom function in Drill we can talk to the same data using a much simpler query.

To get started making user defined functions (UDFs), you first need to download and install [Apache
Maven](https://maven.apache.org/download.cgi). Once you have the tarball, move it to your home directory, and then do:

```
tar xzvf apache-maven-3.3.3-bin.tar.gz
mv apache-maven-3.3.3 apache-maven
```

You should also put Maven's `bin` directory in your system path, so add the line

```
export PATH=$PATH:~/apache-maven/bin
```

to the end of your `.bashrc` or `.bash_profile` file. Next we need to create a new project in Maven for our custom
function. You can do this by issuing a command like:

```
mvn archetype:generate -DgroupId=com.yourgroupidentifier.udf -DartifactId=gahelper -DinteractiveMode=false
```

in whatever folder you prefer to store the project. We're going to call our Google Analytics helper function GAHELPER(),
although you're free to call it whatever you prefer (in fact, FRANK() has a nice ring to it). Now cd to the 'gahelper'
(or 'frank'!) directory, and open up the configuration file for the project: `pom.xml`. Add the following new dependecy
under the `<dependencies>` XML tag:

```
<dependency>
  <groupId>org.apache.drill.exec</groupId>
  <artifactId>drill-java-exec</artifactId>
  <version>1.2.0</version>
</dependency>
```

Then place this block one level under the main `<project>` tag:

```
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-source-plugin</artifactId>
      <version>2.4</version>
      <executions>
        <execution>
          <id>attach-sources</id>
          <phase>package</phase>
          <goals>
            <goal>jar-no-fork</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

Before we start messing with the actual code of the project, we need to do one last thing so that Drill knows
to treat what we're making as a function we can use in queries. Make this folder from within the base directory of your
project (the one that has `pom.xml` in it):

```
mkdir ./src/main/resources
```

And then create the file `./src/main/resources/drill-module.conf`, and fill it with this configuration text:

```
drill {
  classpath.scanning {
    packages : ${?drill.classpath.scanning.packages} [
      com.yourgroupidentifier.udf
    ]
  }
}
```

Okay: Time to check out some UDF source code! Go to the project's `src/main/java/com/yourgroupidentifier/udf`
directory, delete `App.java`, and create a new file named `gaHelper.java`. Paste in the following source (note that the
package name listed here should be the same as the one given in the `drill-module.conf` file we just made):

```java
// This software is covered under the Apache 2.0 license
// Read here: http://www.apache.org/licenses/LICENSE-2.0

package com.yourgroupidentifier.udf;

import org.apache.drill.exec.expr.DrillSimpleFunc;

import org.apache.drill.exec.expr.annotations.FunctionTemplate;
import org.apache.drill.exec.expr.annotations.Param;
import org.apache.drill.exec.expr.annotations.Output;

import org.apache.drill.exec.vector.complex.reader.FieldReader;

import io.netty.buffer.DrillBuf;
import org.apache.drill.exec.vector.complex.writer.BaseWriter;

import javax.inject.Inject;

@FunctionTemplate(name="gahelper", scope=FunctionTemplate.FunctionScope.SIMPLE, nulls=FunctionTemplate.NullHandling.NULL_IF_NULL)

public class gaHelper implements DrillSimpleFunc {

  @Param FieldReader columnsArray;
  @Param FieldReader rowArray;

  @Output BaseWriter.ComplexWriter outWriter;

  @Inject DrillBuf outBuffer;

  public void setup() {}

  public void eval() {

    org.apache.drill.exec.vector.complex.writer.BaseWriter.MapWriter gaMapWriter = outWriter.rootAsMap();

    // Index used to iterate over the 'rows' entries
    Integer i = 0;

    // Work through 'columnHeaders' list, lining it up with the fields in a 'rows' entry
    while(columnsArray.next()) {

      // Pull name and type information about the column
      String colNameString = columnsArray.reader("name").readText().toString();
      String colTypeString = columnsArray.reader("dataType").readText().toString();

      // And the corresponding entry from the 'rows' list
      String rowString = rowArray.readText(i).toString();

      // Save data to the map according to the type indicated in 'columnHeaders'
      if (colTypeString.equals("INTEGER")) {

        org.apache.drill.exec.expr.holders.IntHolder intHolder = new org.apache.drill.exec.expr.holders.IntHolder();

        intHolder.value = Integer.parseInt(rowString);

        gaMapWriter.integer(colNameString).write(intHolder);
      }
      else if (colTypeString.equals("TIME") || colTypeString.equals("FLOAT") || colTypeString.equals("PERCENT") || colTypeString.equals("CURRENCY")) {

        org.apache.drill.exec.expr.holders.Float8Holder floatHolder = new org.apache.drill.exec.expr.holders.Float8Holder();

        floatHolder.value = Float.parseFloat(rowString);

        gaMapWriter.float8(colNameString).write(floatHolder);
      }
      // If it's not one of these, just treat it as a "STRING"
      else {

        org.apache.drill.exec.expr.holders.VarCharHolder rowHolder = new org.apache.drill.exec.expr.holders.VarCharHolder();

        byte[] rowStringBytes = rowString.getBytes();

        outBuffer.reallocIfNeeded(rowStringBytes.length);
        outBuffer.setBytes(0, rowStringBytes);

        rowHolder.start = 0;
        rowHolder.end = rowStringBytes.length;
        rowHolder.buffer = outBuffer;

        gaMapWriter.varChar(colNameString).write(rowHolder);
      }

      i++;
    }
  }
}
```

A more detailed explanation of UDF source code is available in the [Drill
documentation](https://drill.apache.org/docs/tutorial-develop-a-simple-function/), but here are the basics: The '@Param'
annotations indicate variable types for the first and second inputs to the function, and '@Output' does the same for
(you guessed it) the output. After we set up a Drill buffer with the '@Inject' line, '@FunctionTemplate()' specifies
some function behavior, including the name we'll use to invoke it from the command line interface. For this UDF we don't
need to do anything in 'setup(),' but we do need to fill in 'eval()' with code that defines the operation of the
function.

In the case of GAHELPER(), the function processes the first arugment (the 'columnHeaders' entry of the Google Analytics
JSON), along with the second argument (the 'rows' entry), to output a map of keys and values for each row of the GA
data. Along the way it also determines the variable type from 'columnHeaders' so we can, for example, immediately talk
to integers in the output map like they *are* integers instead of having to pass them through a CAST() like we did in the
last article.

Now build the source by running `mvn clean package` in the directory that contains `pom.xml`, and then copy the
resulting files to the Drill install:

```
cp target/*.jar ~/apache-drill/jars/3rdparty
```

Now launch `drill-embedded` and give it a try. The output of the function (truncated somewhat by hand) has a structure like:

```
SELECT GAHELPER(ga_table.columnHeaders,FLATTEN(ga_table.`rows`)) FROM `s3-ga`.`google_analytics_dev_2015-10-19.json`
ga_table;
+--------+
| EXPR$0 |
+--------+
| {"ga:browser":"Chrome","ga:browserVersion":"39.0.2171.71","ga:screenResolution":"1024x768","ga:deviceCategory":"desktop",...} |
| {"ga:browser":"Chrome","ga:browserVersion":"39.0.2171.71","ga:screenResolution":"1024x768","ga:deviceCategory":"desktop",...} |
| {"ga:browser":"Chrome","ga:browserVersion":"40.0.2214.111","ga:screenResolution":"(not set)","ga:deviceCategory":"desktop",...} |
| ... |
```

Now remember that using only standard Drill functions, the query I ended on last time looked like this:

```sql
  SELECT `Date`, `Hour`, Browser, Version, Resolution, Device, SUM(CAST(Sessions AS INTEGER)) `Total Sessions`
    FROM (SELECT data[4] `Date`, data[5] `Hour`, data[0] Browser, data[1] Version, data[2] Resolution, data[3] Device, data[8] Sessions
            FROM (SELECT FLATTEN(ga_table.`rows`) data
                    FROM `s3-ga`.`google_analytics_dev_2015-10-19.json` ga_table))
GROUP BY `Date`, `Hour`, Browser, Version, Resolution, Device
ORDER BY `Date`, `Hour`;
```

But if we use the awesome power of the GAHELPER() UDF, we can perform the same query with:

```sql
  SELECT m.r.`ga:date` `Date`, m.r.`ga:hour` `Hour`, m.r.`ga:browser` Browser, m.r.`ga:browserVersion` Version,
         m.r.`ga:screenResolution` Resolution, m.r.`ga:deviceCategory` Device, SUM(m.r.`ga:sessions`) `Total Sessions`
    FROM (SELECT GAHELPER(ga_table.columnHeaders,FLATTEN(ga_table.`rows`)) AS r
            FROM `s3-ga`.`google_analytics_dev_2015-10-19.json` ga_table) AS m
GROUP BY m.r.`ga:date`, m.r.`ga:hour`, m.r.`ga:browser`, m.r.`ga:browserVersion`, m.r.`ga:screenResolution`, m.r.`ga:deviceCategory`
ORDER BY m.r.`ga:date`, m.r.`ga:hour`;
```

Note the advantages here! We only need *one* subquery now, and we can reference the data directly using intrinsic Google
Analytics names ('ga:browserVersion', etc.). In fact, by eliminating the need to manually determine what each index
means ('4' for 'Date', or '0' for 'Browser') we're skipping a whole step that's implicit in the first query. And don't
forget that now we can call SUM() directly on the sessions count without a preceeding CAST() since GAHELPER() is smart
enough to determine the types of values.

So that's my whirlwind tour of the custom function capability of Apache Drill. Time to start thinking about how a UDF
could improve your data experience!
