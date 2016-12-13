+++
type = "blog"
author = "Nathan Griffith"
categories = ["drill", "mongodb", "cluster", "sharded", "database", "sql"]
date = "2015-12-02"
title = "Setting up an SQL interface to a sharded MongoDB"

+++

Salutations, data fans! Today I'd like to demonstrate how to use Apache Drill to query a sharded MongoDB database
running on a computer cluster.

To begin with, I'm going to assume you already know how to do a sharded Mongo setup (the [official
documentation](https://docs.mongodb.org/manual/sharding/) is very helpful), or you already have access to a system with
a sharded MongoDB in place. With this out of the way, I can focus on the Drill setup.

When running in a multiple machine environment, Drill can increase its efficiency for very large queries by coordinating
between several instances of a service called a 'drillbit.' To help track these instances we use a piece of software
called [Apache Zookeeper](https://zookeeper.apache.org/). Once Zookeeper is installed, you should make a 'zoo.cfg' file
in its './conf' directory. This simple one works fine for our purposes (create a 'dataDir' as needed):

```
tickTime=2000
dataDir=/home/cluster-user/zookeeper/data
clientPort=2183
```

Next you should download and install Drill to each of the nodes in the cluster that you'd like a drillbit to run on. For
my setup I've chosen to run a drillbit on each node that contains a MongoDB shard. This is a good idea because Drill
considers the location of data stores when it performs queries. The ./conf/drill-override.conf file for each install
should be edited to reflect the IP address and port of your Zookeeper. For instance, you could write:

```
drill.exec: {
  cluster-id: "drill-cluster-example",
  zk.connect: "172.31.0.0:2183"
}
```

Where the name "drill-cluster-example" is the same for each node.

With this minor bit of configuration out of the way, all that's left is to start the software. First bring up the
Zookeeper server with '~/zookeeper/bin/zkServer.sh start', and then launch the drillbits by running
'~/apache-drill/bin/drillbit.sh start' on each node.
After you've started everything, you can run 'drill-conf' to bring you to an SQL prompt. If all is well, you
should be able to run `SELECT * FROM sys.drillbits;` and see each drillbit on the cluster report in. For example on a
three machine test cluster I saw the following:

```
> select * from sys.drillbits;
+----------------------------------------------+------------+---------------+------------+----------+
|                   hostname                   | user_port  | control_port  | data_port  | current  |
+----------------------------------------------+------------+---------------+------------+----------+
| ip-172-31-41-122.us-west-2.compute.internal  | 31010      | 31011         | 31012      | false    |
| ip-172-31-41-120.us-west-2.compute.internal  | 31010      | 31011         | 31012      | false    |
| ip-172-31-41-121.us-west-2.compute.internal  | 31010      | 31011         | 31012      | true     |
+----------------------------------------------+------------+---------------+------------+----------+
```

Now all that's left is to activate the Drill interface to MongoDB. Open the Drill Web Console by visiting one of the
nodes that's running a drillbit (use port 8047, just like before). From here click "Enable" next to the "mongo" plugin,
and then hit "Update." Now change "localhost:27017" in the "connection" field to correspond to the IP and port of the
machine that runs your 'mongos' process. If you have more than one "mongos" you can specify them all like this:

```
...
  "connection": "mongodb://172.31.41.120:27017,172.31.41.121:27017,172.31.41.122:27017",
...
```

And that's it! You're now all set to run SQL queries on your sharded MongoDB via Drill. For an example of how to
interface with some complicated JSON-based data that's typical of MongoDB collections, check out [this
other](http://www.dremio.com/blog/bless-this-mess-working-with-complicated-json-structure-in-sql/) post.
