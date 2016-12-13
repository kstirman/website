+++
type = "blog"
author = "Nathan Griffith"
categories = ["drill", "cloudera", "cdh", "kerberos", "aws", "ec2", "sql", "hadoop"]
date = "2016-01-25"
title = "Securing SQL on Hadoop, Part 2: Installing and configuring Drill"
+++

Today we're going to pick up where we left off in [Part
1](http://www.dremio.com/blog/securing-sql-on-hadoop-part-1-installing-cdh-and-kerberos/) of my two-parter about setting
up a CDH cluster to perform secure SQL queries on an HDFS store. As you recall, last time we had just finished using
Cloudera Manager's wizard to finalize a Kerberos configuration, and all of the cluster services had come back online
using the new security system so our basic HDFS cluster set-up was good to go. All that's left to do now is install and
configure the piece of software that implements the SQL interface: Apache Drill.

Step 1.) Drill prerequisites
----------------------------

First things first: We're gonna need Java 7 or better on our CDH machines. Following some useful information found
[here](http://lifeonubuntu.com/ubuntu-missing-add-apt-repository-command/) and
[here](http://askubuntu.com/questions/508546/howto-upgrade-java-on-ubuntu-14-04-lts), we can set up a new package
repository and pull it down using:

```
$ sudo apt-get install software-properties-common python-software-properties
$ sudo add-apt-repository ppa:webupd8team/java
$ sudo apt-get update
$ sudo apt-get install oracle-java7-installer
```

(Remember to do this for each node on your cluster!)

Step 2.) Install Drill
----------------------

Time to install Drill and tell it where the cluster's 'Zookeeper' is (REMEMBER: Just like installing Java, this step
needs to be done on every node). Start by downloading and unpacking the software:

```
$ wget http://apache.osuosl.org/drill/drill-1.4.0/apache-drill-1.4.0.tar.gz
$ tar xzvf apache-drill-1.4.0.tar.gz
```

Next edit Drill's `conf/drill-override.conf` file so that it looks like:

```
drill.exec: {
  cluster-id: "drill-cdh-cluster",
  zk.connect: "<ZOOKEEPER IP>:2181"
}
```

Where 'ZOOKEEPER IP' should be changed to the IP of the CDH machine that runs the zookeeper process (this is probably
the same as the Main Node from the last article).

If you'd like, you can add Drill to the system path by adding this to `.bashrc`:

```
export PATH=$PATH:/apache-drill-1.4.0/bin
```

Step 3.) Configuring Drill for HDFS
-----------------------------------

Now we need to set up Drill so that it can read our cluster's HDFS. Open the Drill Web Console by going to
http://&lt;MAIN NODE IP>:8047. Click 'Storage' at the top and then 'Update' next to the dfs plugin and copy the JSON
that you find in the text field. Next make a new storage plugin called 'hdfs' (previous page) and then paste in the text
you just copied, replacing the 'null' that's already there.

We'll proceed by making a small modification that turns this standard 'dfs' plugin in into our new one for HDFS.

Take the line

```
"connection": "file:///",
```

and replace it with

```
"connection": "hdfs://<ADDRESS OF HDFS NAMENODE>:8020",
```

Hit "Create" and now Drill is ready to read your HDFS data!

Step 4.) Enabling Kerberos support for Drill
--------------------------------------------

So now we have HDFS, Kerberos, *and* Drill, but currently Drill can't talk to the HDFS we have running because it
requires authentication. Let's fix that.

First we should make an HDFS superuser account as indicated in this [Cloudera
doc](http://www.cloudera.com/documentation/enterprise/latest/topics/cm_sg_s5_hdfs_principal.html). On the Main Node, run
`sudo kadmin.local` and add an 'hdfs' principal with this command:

```
addprinc hdfs@KERBEROS.CDH
```

(Hit Ctrl-d to exit the prompt). In order to enable authentication with Kerberos, we also need to copy the file
`hadoop-yarn-api.jar` into Drill's class path:

```
$ cp /opt/cloudera/parcels/CDH-5.5.1-1.cdh5.5.1.p0.11/lib/hadoop/client/hadoop-yarn-api.jar ~/apache-drill/jars/
```
(NOTE: *The above step and the three following must be performed on each node of the cluster*.)

Next, Drill's `conf/core-site.xml` file should be edited to contain the following snippet of xml:

```
<property>
  <name>hadoop.security.authentication</name>
  <value>kerberos</value>
</property>
```

All that's left to do is create an 'hdfs' Kerberos ticket for the user that we're logged into

```
$ kinit hdfs@KERBEROS.CDH
```

and then start up each of the drillbits

```
drillbit.sh start
```

So now Drill has both the configuration and the authority to use our kerberized HDFS store. Give it a shot by opening up
a Drill prompt (`drill-conf`) and trying a query.

For example, here's a test query on my handy 250 GB of Reddit data:

```
> SELECT title, TO_TIMESTAMP(CAST(created_utc AS INT)) created, score FROM hdfs.`/data/RS_full_corpus.json` WHERE over_18 = false AND score >= 100 LIMIT 10;
+------------------------------------------------------------------------------+------------------------+--------+
|                                    title                                     |        created         | score  |
+------------------------------------------------------------------------------+------------------------+--------+
| Early Retirement Guide - Phillip Greenspun                                   | 2006-01-30 19:21:17.0  | 223    |
| Programming Like A Mathematician I: Closures                                 | 2006-01-29 18:14:30.0  | 178    |
| More than 1000 wikipedia alterations by US Representative Staffers           | 2006-01-29 12:57:45.0  | 304    |
| Great Design: What is Design?                                                | 2006-01-26 20:54:30.0  | 143    |
| Use Python (not Java) to teach programming                                   | 2006-01-26 19:43:23.0  | 167    |
| The Demotivators!                                                            | 2006-01-26 18:02:51.0  | 168    |
| Just how much can you achieve with pure CSS? Some amazing demonstrations.    | 2006-02-26 18:07:57.0  | 329    |
| A summary of two lectures by Alan Kay                                        | 2006-02-24 04:56:49.0  | 110    |
| How Intel could buy Hollywood and profit by selling more DRM-less machines.  | 2006-02-20 20:48:18.0  | 108    |
| "zeitgeist" reddits, covering popular topics, eg, the infamous cartoons      | 2006-02-19 19:33:59.0  | 299    |
+------------------------------------------------------------------------------+------------------------+--------+
```

And that wraps up Part 2 of this how-to guide. May all your queries be fruitful and secure!

*Acknoweldgements: Many thanks to William Witt over on the user@drill.apache.org mailing list for providing crucial
information about Kerberos-Drill configuration!*
