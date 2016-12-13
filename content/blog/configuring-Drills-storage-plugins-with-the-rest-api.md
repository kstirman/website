+++
type = "blog"
author = "Nathan Griffith"
categories = ["drill", "rest", "storage plugin", "plugins", "browser", "console"]
date = "2015-12-02"
title = "Configuring Drill's storage plugins with the REST API"

+++

The Web Console of Apache Drill is an intuitive configuration tool, but it's easy to imagine scenarios where connecting 
to a Drill instance via a browser isn't desirable or even possible (e.g. within a script or in a high security
environment).

So if you find yourself needing to set up a Drill storage plugin in such a situation, what should you do? Well, in cases
like this Drill's built-in REST API comes to the rescue.

As demonstrated in the following Python script, setting up a new plugin (in this case one for HDFS) is a pretty simple
operation when performed via the REST interface:

```python
import httplib
import json

# Create a connection to the local instance of Drill
drillRestConn = httplib.HTTPConnection("localhost:8047")

# First GET standard 'dfs' plugin via REST
drillRestConn.request("GET","/storage/dfs.json")
drillRestResponse = drillRestConn.getresponse()
storageDict = json.loads(drillRestResponse.read())

# Now modify it into one for HDFS
storageDict["name"] = "hdfs"
storageDict["config"]["connection"] = "hdfs://<ADDRESS OF HDFS NAMENODE>:8020"
storageJson = json.dumps(storageDict)

# Then POST it back via REST
pluginHeader = {"Content-Type": "application/json"}
drillRestConn.request("POST", "/storage/hdfs.json", storageJson, pluginHeader)
drillRestReponse = drillRestConn.getresponse()
```

(This code is based on a portion of the EMR bootstrap script indicated in [this
post](http://www.dremio.com/blog/bootstrapping-an-interactive-sql-environment-on-amazon-emr/).)

Notice how the first endpoint I use is labeled by the name of an existing plugin that I'd like information about (in
this case 'dfs' which is the name of the default plugin for the local filesystem). This information is returned in the
form of a JSON string which starts out like:

```
{
  "name" : "dfs",
  "config" : {
    "type" : "file",
    "enabled" : true,
    "connection" : "file:///",
    "workspaces" : {
      "root" : {
        "location" : "/",
        "writable" : false,
        "defaultInputFormat" : null
      },
      "tmp" : {
        "location" : "/tmp",
        "writable" : true,
        "defaultInputFormat" : null
      }
    },
    ... etc
```

To change this plugin to reference an HDFS store, all I have to do is edit the JSON so that the "name" and then the
"connection" key (located under "config") have appropriate values. Once that's done I can POST the modified
configuration JSON back to Drill, using the name of the new plugin that I'd like to create as the label of the endpoint.

So if you find yourself browserless and in need of some Drill plugin configuration&mdash;don't despair! You're only a
couple REST requests away from a console-based solution.
