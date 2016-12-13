+++
type = "blog"
author = "Nathan Griffith"
categories = ["drill", "cluster", "sql", "amazon", "aws", "emr", "hadoop"]
date = "2015-11-17"
title = "Bootstrapping an interactive SQL environment on Amazon EMR"

+++

Amazon's Elastic MapReduce (EMR) service is a great way to create on-demand Hadoop clusters for ad hoc data analysis.
The web based interface for the service is simple to use, and at its most basic level creating an EMR cluster is as easy
as specifying the number and type of machines to use. But you can also instruct each computer to execute something
called a 'bootstrap action' just before coming online. The contents of this script can direct the cluster to
automatically prepare itself for data crunching operations that have dependencies which lie outside of the primary
Hadoop toolset. In this post I'll show you how to use an EMR bootstrap action provided by Dremio to install Drill on
each node in the cluster. As I've discussed before, Drill is great for a lot of stuff, and it's a fast and easy way to
get near-realtime access to your data via a SQL interface.

Launching a new EMR cluster that comes with Drill pre-installed is almost identical to launching one without it. The
only extra step is that you need to specify the location on Amazon S3 of a bootstrap action script. You can do this
under the "Bootstrap Actions" heading of the EMR "Advanced Options" page for cluster creation (it's the second settings
grouping from the bottom of the page). Just select "Custom action" from the dropdown list, press "Configure and add",
and then specify Dremio's script for installing Drill (hosted at s3://repo.dremio.com/aws/emr/bootstrapDrill.py) in the
"JAR location" field.

Once your cluster is up and ready, you should be able to log in and run `drill-conf` to start Drill's SQL prompt. Type
`SELECT * FROM sys.drillbits;` and you'll see all of the available drillbit services on your cluster (one for each
machine) listed like the following:

```
> SELECT * FROM sys.drillbits;
+---------------------------------------------+------------+---------------+------------+----------+
|                  hostname                   | user_port  | control_port  | data_port  | current  |
+---------------------------------------------+------------+---------------+------------+----------+
| ip-172-31-23-93.us-west-2.compute.internal  | 31010      | 31011         | 31012      | true     |
| ip-172-31-26-64.us-west-2.compute.internal  | 31010      | 31011         | 31012      | false    |
| ip-172-31-23-94.us-west-2.compute.internal  | 31010      | 31011         | 31012      | false    |
+---------------------------------------------+------------+---------------+------------+----------+
```

Since this script comes with the HDFS plugin for Drill pre-installed, you can start querying Hadoop data right away with
commands like `` SELECT * FROM hdfs.`/data/userdata.json` ``.

You can also enable access to files stored on Amazon S3 by specifying the name of an S3 bucket (without a leading
"s3://") in the "Optional arguments" box of the "Add Bootstrap Action" dialog of the EMR web GUI. The name for this
plugin will be "s3," so you can query a file in the bucket you specified with just `` SELECT * FROM s3.`myfile.csv` ``.

So if you're curious about using Drill in a cluster, pairing Amazon's EMR service with this bootstrap action could be a
great place to start.
