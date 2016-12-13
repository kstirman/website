+++
author = "Nathan Griffith"
categories = ["drill", "sql", "udfs", "custom functions", "java", "sentiment", "anaylsis"]
date = "2016-01-28"
title = "Writing a custom SQL function for sentiment analysis"
type = "blog"
+++

In the world of data analytics a 'sentiment analysis' is any technique that attempts to represent the feelings of users
in a somewhat quantitative way. Implementations of this idea vary, but one of the simplest ones involves giving
individual words a numeric score according to the strength of the positive or negative the emotions that they elicit.
For instance we might assign a score of -2.3 to the word 'disappointment' and a score of 1.8 to 'lighthearted.'

In today's article I'm going to demonstrate that writing a custom SQL function (also known as a user defined function,
or UDF) that performs a sentiment analysis is a fairly straightforward task. The SQL platform we'll be using for this
project is Apache Drill; A software capable of querying *many* different types of data stores that also allows for the
creation of custom functions written in the Java programming language.

This isn't the first time I've written about creating a UDF for Drill, and readers looking for a richer set of
information about UDF programming may want to refer to [this earlier
article](http://www.dremio.com/blog/querying-google-analytics-json-with-a-custom-sql-function/).

Downloading Maven and starting a new project
--------------------------------------------

Just as before, we'll want to start by downloading and installing Apache Maven ([available
here](https://maven.apache.org/download.cgi)), which will be responsible for managing and building our Java project:

```
$ tar xzvf apache-maven-3.3.9-bin.tar.gz
$ mv apache-maven-3.3.9 apache-maven
```

And since you'll probably be using it fairly frequently it might be nice to put the Maven binary in the PATH environment
variable, so add this line to your `.bashrc` file:

```
export PATH=$PATH:~/apache-maven/bin
```

Now go to whatever directory you'd like to store your UDFs in and issue this Maven command to create a new project for
our sentiment analyzer called 'simplesentiment':

```
$ mvn archetype:generate -DgroupId=com.dremio.app -DartifactId=simplesentiment -DinteractiveMode=false
```

Because our UDF relies on Apache Drill, we'll need to add a couple things to the project's `pom.xml` configuration file.
The first should go within the `<dependencies>` tag:

```
<dependency>
  <groupId>org.apache.drill.exec</groupId>
  <artifactId>drill-java-exec</artifactId>
  <version>1.4.0</version>
</dependency>
```

and the next should go inside the outermost tag called `<project>`:

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

Finally we need to make a `./src/main/resources/drill-module.conf` file for the project (you'll probably need to create
the 'resources' directory, so go ahead and do that). This file should have these contents:

```
drill {
  classpath.scanning {
    packages : ${?drill.classpath.scanning.packages} [
      com.yourgroupidentifier.udf
    ]
  }
}
```

Where `com.yourgroupidentifier.udf` should be the same name as the `package` specified in the Java files listed in the
next section.

Sentiment analysis UDF source code
----------------------------------

The sentiment analyzer in my UDF follows the simple algorithm that I described earlier, with the values for words
provided by [this
file](https://github.com/cjhutto/vaderSentiment/blob/master/build/lib/vaderSentiment/vader_sentiment_lexicon.txt)
(`vader_sentiment_lexicon.txt`) available on Github from user 'cjhutto.'

Because Drill is picky about the format of a UDF class, this custom function had to be expressed in two different source
files: one to define all the function's operations, and another for a simple class to hold the dictionary that
translates words to numeric sentiment values. For this project, these files will be located in the project's
`main/java/com/yourgroupidentifier/udf` directory.

The first file, `simpleSent.java` looks like:

```java
package com.yourgroupidentifier.udf;

import org.apache.drill.exec.expr.DrillSimpleFunc;
import org.apache.drill.exec.expr.holders.Float8Holder;
import org.apache.drill.exec.expr.holders.NullableVarCharHolder;

import org.apache.drill.exec.expr.annotations.FunctionTemplate;
import org.apache.drill.exec.expr.annotations.Output;
import org.apache.drill.exec.expr.annotations.Param;

@FunctionTemplate(
        name = "simplesent",
        scope = FunctionTemplate.FunctionScope.SIMPLE,
        nulls = FunctionTemplate.NullHandling.NULL_IF_NULL
)

public class simpleSent implements DrillSimpleFunc {

    // The input to the function---almost certainly a text field
    @Param
    NullableVarCharHolder input;

    // The output of the function---just a number
    @Output
    Float8Holder out;

    public void setup() {

        // Initialize object that holds dictionary
        new com.yourgroupidentifier.udf.dictHolder();

        // Open the sentiment values file
        try {
            java.io.FileReader fileReader = new java.io.FileReader("/path/to/vader_sentiment_lexicon.txt");
            java.io.BufferedReader bufferedReader = new java.io.BufferedReader(fileReader);

            String currLine;

            // Read each line
            try {
                while ((currLine = bufferedReader.readLine()) != null) {
                    String[] splitLine = currLine.split("\\s+");

                    String currWord = splitLine[0];
                    Double currValue;
                    try {
                        currValue = Double.parseDouble(splitLine[1]);
                    }
                    catch (java.lang.NumberFormatException numberEx) {
                        currValue = 0.0;
                    }

                    // Put sentiment value in dictionary
                    com.yourgroupidentifier.udf.dictHolder.sentiDict.put(currWord, currValue);
                }
            }
            catch(java.io.IOException ioEx) {
                System.out.print("IOException encountered");
            }

        }
        catch(java.io.FileNotFoundException fileEx) {
            System.out.println("Sentiment valences file not found!");
        }
    }

    public void eval() {

        // Initialize output value
        out.value = 0.0;

        // Split up the input string
        String inputString = org.apache.drill.exec.expr.fn.impl.StringFunctionHelpers.toStringFromUTF8(input.start, input.end, input.buffer);
        String[] splitInputString = inputString.split("\\s+");

        for(int i = 0; i < splitInputString.length; i++) {

            java.lang.Object result = com.yourgroupidentifier.udf.dictHolder.sentiDict.get(splitInputString[i].toLowerCase());

            if(result != null) {

                Double wordValue = ((Double) result);

                out.value += wordValue;
            }
        }

    }

}
```

(Remember to change the line with `/path/to/vader_sentiment_lexicon.txt` so that it reflects the location of the file on
your system!)

The second file is called `dictHolder.java`, and contains this small class:

```java
package com.yourgroupidentifier.udf;

public class dictHolder {
    static public java.util.Hashtable<String, Double> sentiDict;

    public dictHolder() {
        sentiDict = new java.util.Hashtable<String, Double>();
    }
}
```

Building and installing our UDF
-------------------------------

To build and install the custom function, just go to the project's root directory (the one with `pom.xml`) and issue
these commands

```
$ mvn clean package
$ cp target/*.jar ~/apache-drill/jars/3rdparty
```

changing the second command to be appropriate for your Drill install.

And that's literally all there is to it! You should now be able to invoke the `SIMPLESENT()` function from within
Drill's SQL prompt. In the next article I'll be doing exactly that as I explore a corpus of Reddit submission titles
using this handy new analysis tool.
