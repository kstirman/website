+++
type = "blog"
author = "Nathan Griffith"
categories = ["windows", "microsoft", "drill", "install"]
date = "2015-12-08"
title = "Installing Apache Drill on Microsoft Windows"

+++

Many of my previous articles on the Dremio Blog have assumed that you'd be using Apache Drill from within some flavor of
unix (namely either Linux or OS X), but it's completely possible to install Drill on Microsoft Windows as well. In this
post I'll cover how to install and run a single-machine instance of Drill on Windows Server 2012 R2.

Just like the unix versions, you'll need to have the Java Development Kit installed to run Drill. You can grab JDK
version 7 from [this site](http://www.oracle.com/technetwork/java/javase/downloads/jdk7-downloads-1880260.html). You'll
also need some way to uncompress the `.tar.gz` file extension that Drill comes packaged in. The open source Windows
utility [7-Zip](http://www.7-zip.org/) is a popular choice for this task. (Note: You will need to 'unzip' the Drill file
twice&mdash;once to kill the '.gz' extension and another time to get rid of the '.tar').

Before we run Drill we need to set up some environment variables in Windows. In Windows Server 2012, you can edit these
by opening the Control Panel, hitting 'System and Security', 'System,' and then 'Advanced system settings' on the
left. Finally, head to the 'Advanced' tab in this new window and click the 'Environment Variables...' button.

<br>
<br>
<p style="text-align: center;">
<img style="max-width: 100%;" src="/img/windows_env.png">
</p>
<p style="text-align: center; font-style: italic;">Configuring environment variables in Windows Server 2012.</p>
<br>
<br>

Now we need to create a new user variable, so hit 'New...' under the first list and put in a variable name of
`JAVA_HOME` with a value that indicates the directory of your Java install (in my case: 'C:\Program
Files\Java\jdk1.7.0_79'). Next we'll edit the value of the `Path` variable in the 'System variables' list by adding a
semi-colon followed by 'C:\Program Files\Java\jdk1.7.0_79\bin'.

Almost there! Open a Command Prompt window and cd to the 'bin' directory where you unpacked Drill. Now type the
following command:

```
sqlline.bat -u "jdbc:drill:zk=local"
```

and you should be presented with a standard single-machine Drill session.

Drill is just as easy to use on Windows as it is on unix, and you can look forward to more Windows-focused content on
the Dremio Blog real soon!
