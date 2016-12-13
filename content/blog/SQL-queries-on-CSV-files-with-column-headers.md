+++
title = "SQL queries on CSV files with column headers"
type = "blog"
author = "Nathan Griffith"
categories = ["drill", "csv", "column headers", "sql", "files"]
date = "2015-11-23"

+++

A new feature of Drill 1.3.0 is the ability to query CSV files using the column names listed in the file's header.

As an example, let's look at [this CSV formatted file](http://introcs.cs.princeton.edu/java/data/olympic-medals2012.csv)
of countries that participated in the 2012 Summer Olympics. Before we begin, rename the file so that it ends in '.csvh'
instead of '.csv'. The extension change tells Drill to parse the header of the CSV file, which means we can use the
in-file identifiers for column names when we construct our queries.

So now we can ask for the 10 countries with the most medals by entering:

```
> SELECT `Country.name` AS Country, CAST(Total AS INT) `Total Medals` FROM dfs.`/path/to/olympic-medals2012.csvh` ORDER BY `Total Medals` DESC LIMIT 10;
+--------------+---------------+
|   Country    | Total Medals  |
+--------------+---------------+
| US           | 104           |
| China        | 88            |
| Russia       | 82            |
| UK           | 65            |
| Germany      | 44            |
| Japan        | 38            |
| Australia    | 35            |
| France       | 34            |
| Italy        | 28            |
| South Korea  | 28            |
+--------------+---------------+
```

We can also enable header parsing for any extension (including regular .csv files) by using the Drill Web Console to
edit the JSON of a storage plugin. You can open the console by running 'drill-embedded' and going to
http://localhost:8047 in a browser. Once you've loaded the page, go to "Storage" at the top and click "Update" next to
(for example) the 'dfs' plugin for the local file system. To enable header parsing for a file type we need to add the
key-value pair `"extractHeader": true` to an entry in the "formats" section of the file. For instance, if we change the
plugin so that tab separated (.tsv) files now have their headers read, then the section for that format would read:

```
...
    "tsv": {
      "type": "text".
      "extensions": [
        "tsv"
      ],
      "extractHeader": true,
      "delimiter": "\t"
    },
...
```

Drill was *already* an amazing tool for querying text files. But header parsing makes it even better!
