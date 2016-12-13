+++
type = "blog"
author = "Nathan Griffith"
categories = ["drill", "kudu", "sql", "apache incubator"]
date = "2016-01-13"
title = "SQL on Kudu quickstart"
+++

[Kudu](http://getkudu.io/) is a new (currently in beta) distributed data storage system that attempts to unify the
strengths of HBase and HDFS systems. Although initially developed at Cloudera, the software has recently been inducted
into the Apache Incubator program. This means there's a reasonable expectation of Kudu becoming the next big Next Big
Thing in terms of data storage.

As the widely varied topics of this blog demonstrate, Drill devs aren't very willing to let a potential source of
valuable data slip by unnoticed, so of course they've been hard at work on a new storage plugin. Starting in Drill 1.5
(due out January 2016), performing SQL queries on a Kudu system will be as easy as:

1.) [Installing Drill](https://drill.apache.org/docs/installing-drill-on-linux-and-mac-os-x/)

2.) Starting the `drill-embedded` binary

3.) Opening the Drill Web Console (http://localhost:8047)

4.) Navigating to 'Storage' and creating a new 'kudu' plugin that contains this configuration:

```
{
  "type": "kudu",
  "enabled": true,
  "masterAddresses": "<kudu_master_server_ip_address_1>,<kudu_master_server_ip_address_2>,<etc>"
}
```

5.) Enabling the storage plugin

You should now see 'kudu' show up after running a `show databases;` command within Drill.
