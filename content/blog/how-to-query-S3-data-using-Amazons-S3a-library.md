+++
type = "blog"
author = "Nathan Griffith"
categories = ["aws", "s3", "s3a", "drill", "sql"]
date = "2015-11-16"
title = "How to query S3 data using Amazon's S3a library"

+++

Starting with version 1.3.0, Drill has the ability to query files stored on Amazon's S3 cloud storage using the S3a
library. This is important, because S3a adds support for files bigger than 5 gigabytes (these were unsupported using
Drill's previous S3n interface).

To enable Drill's S3a support, first edit the file `conf/core-site.xml` in your Drill install directory, replacing
the text ENTER_YOUR_ACESSKEY and ENTER_YOUR_SECRETKEY with your AWS credentials.

```xml
<configuration>

  <property>
    <name>fs.s3a.access.key</name>
    <value>ENTER_YOUR_ACCESSKEY</value>
  </property>

  <property>
    <name>fs.s3a.secret.key</name>
    <value>ENTER_YOUR_SECRETKEY</value>
  </property>

</configuration>
```

Next you'll need to duplicate the 'dfs' plugin in the 'Storage' section of the Drill Web Console, which is located at
`localhost:8047`. (Note: on a single machine system, you'll need to run `drill-embedded` before you can access the web
console site). To do this, hit 'Update' next to 'dfs,' and then copy the JSON text that appears. Now create a new
storage plugin on the previous page, and paste in the 'dfs' text, replacing the text `file:///` with
`s3a://your.bucketname`. It doesn't matter what you named your new plugin, but it might be helpful to reference the s3
and/or the bucket name so you remember what it's for.

And that's it! You should now be able to talk to data stored on S3 using the S3a library.
