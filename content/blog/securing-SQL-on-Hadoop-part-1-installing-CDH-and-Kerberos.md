+++
title = "Securing SQL on Hadoop, Part 1: Installing CDH and Kerberos"
type = "blog"
author = "Nathan Griffith"
categories = ["drill", "cloudera", "cdh", "kerberos", "aws", "ec2", "sql", "hadoop"]
date = "2016-01-22"

+++

Introduction
------------

In the Dremio Blog we've talked about pairing Drill with HDFS before, but never with the emphasis on security that so
often comes hand-in-hand with enterprise applications. So today's post marks the beginning of a two part series
explaining how to set up a cluster environment that enables *secure* SQL queries to data stored on HDFS. For this
article's test 'hardware' we'll use six instances provisioned from Amazon's EC2 service, while the core software
components will be provided by Cloudera's Hadoop system, CDH, paired with Ubuntu 14.04. The 'secure' element of this
cluster environment with be enabled via Kerberos, which is an industry standard in the realm of authentication software.

Step 1.) Cluster Set Up
----------------------

In general I'll be going into a significant amount of detail for this setup, since Kerberos configuration can be, well,
*hard*. I am, however, going to assume right now that you know your way around the AWS Management Console, and can
handle setting up a few instances and configuring them by yourself. Remember to select 'Ubuntu 14.04' for the operating
system, and make a security group for the cluster that opens all the the TCP and UDP ports between machines in the
cluster, just to be sure they can communicate freely with eachother. You'll also want to open all ICMP ports (for ping),
and TCP port 7180 for the Cloudera Manager software that we'll be using shortly. Things like system specs and storage
are obviously somewhat up to you to decide on, but I went with fairly muscular 'm4.xlarge' instances that had 250 GB of
storage each (more than big enough for my Reddit submission corpus test data, which was featured in [this previous
article](http://www.dremio.com/blog/old-and-busted-teasing-formerly-fashionable-websites-from-reddit-data/)).

Step 2.) Install CDH with Cloudera Manager
-----------------------------------------

We'll perform the install of CDH (and thus, our HDFS store) by using Cloudera's awesome Cloudera Manager software, which
is [available here](http://www.cloudera.com/downloads/manager/5-5-1.html).

After you download the .bin file from this site to one of the nodes on your cluster, mark it as executable and run it
(from here on out I'll call the node that you selected for this step the 'Main Node').

```
$ chmod +x cloudera-manager-installer.bin
$ sudo ./cloudera-manager-installer.bin
```

Once this software finishes running you can point a browser to the Main Node's URL (available from the AWS EC2 web
interface) on port 7180. From here you can complete the install of CDH to rest of your cluster (the initial username and
password are both 'admin'). There are various options that show up as you progress through the steps, but for this
simple example it's sufficient to accept the defaults for most cases. Just be sure to check the "Install Oracle Java SE
Development Kit" and "Install Java Unlimited Strength Encryption Policy Files" boxes when they show up (this last one
can be relevant to Kerberos encryption). You'll also need to specify the user 'ubuntu' and the relevant private key
'.pem' file for your cluster when you hit the "Provide SSH login credentials" screen. And the basic "Core Hadoop" option
is fine when t comes to choosing what package set to install.

That's really all there is to getting the Hadoop side of things running for this project. Shockingly easy, right?

But now it's time for Kerberos. Sigh.

Step 3a.) Install and Configure Kerberos: Main Node
-------------------------------------------------

Okay, it's not actually all that bad. All we have to do is install some packages on the cluster and do some small edits
to conf files before we can hand it off to the Clouder Manager wizard for Kerberos configuration (let me say it again:
Cloudera Manager is awesome).

Let's begin with that needs to be done on the Main Node, which in the parlance of Kerberos will become the 'KDC' (Key
Distribution Center) of our authentication system. As per this [Ubuntu
documentation](https://help.ubuntu.com/community/Kerberos) (which was a useful reference for many of the following
steps), go ahead and install these two packages

```
$ sudo apt-get install krb5-kdc krb5-admin-server
```

entering "KEBEROS.CDH" for the realm name in the first field that pops up, and specifying the Main Node's internal DNS
name in the next two (again these can be found on the AWS EC2 instance page&mdash;they're formatted like
"ip-172.XXX.XXX.XXX.us-west-2.compute.internal"). Then run

```
$ sudo dpkg-reconfigure krb5-kdc
```

and select "Yes."


Now it's time to edit the `/etc/krb5kdc/kdc.conf` file as recommended in [these Cloudera
instructions](http://www.cloudera.com/documentation/enterprise/latest/topics/cm_sg_s4_kerb_wizard.html). Place the
lines

```
max_life = 1d
max_renewable_life = 7d
kdc_tcp_ports = 88
```

in the KERBEROS.CDH '[realms]' entry, replacing any similar lines and deleting 'kdc_ports' both there underneath
'[kdcdefaults].' Alright! On to the next file.

Create (or open) `/etc/krb5kdc/kadm5.acl`, and insert this line:

```
*/admin@KERBEROS.CDH    *
```

Now it's time to initialize the Kerberos realm with:

```
$ sudo krb5_newrealm
```

Cool. Now, we need to create the credentials that the Cloudera Manager wizard will use when it completes our Kerberos
setup. To do this enter the command

```
$ sudo kadmin.local
```

and from within the prompt that's presented, type:

```
addprinc -pw <PASSWORD> cloudera-scm/admin@KERBEROS.CDH
```

Now hit Crtl-D to exit the prompt, and install one last package:

```
$ sudo apt-get install ldap-utils
```

Step 3b.) Install and Configure Kerberos: Other Nodes
-----------------------------------------------------

Compared to the Main Node setup, this step is mercifully brief. For each other note in the cluster you just need to
install this package:

```
$ sudo apt-get install krb5-user
```

When you're presented with the text fields this time you can just hit enter, because the wizard will end up handling
this stuff for you in the end.

Step 4.) Use the Cloudera Manager Kerberos wizard
---------------------------------------------------

To start the Kerberos wizard, select "Enable Kerberos" from the dropdown menu for the cluster (see Figure 1).

<br>
<p style="text-align: center;">
<img style="max-width: 100%;" src="/img/cloudera_kerberos.jpg">
</p>
<p style="text-align: center; font-style: italic;"><b>Figure 1</b>: Starting Cloudera Manager's Kerberos wizard.</p>
<br>

First you're asked to check a bunch of boxes which are friendly reminders of all the prep work you need to do before the
wizard can take over. We've done all of this in the previous steps, so you can check away with wild abandon.

On the next page put the AWS internal address for the Main Node in the field marked "KDC Server Host" and then
"KERBEROS.CDH" for the realm name. Hit "Continue" and then check the "Manage krb5.conf through Cloudera Manager" box.
The defaults here are fine.

Now you need to enter the credentials that were set up in Step 3a. As per [this Cloudera
doc](http://www.cloudera.com/documentation/enterprise/latest/topics/cm_sg_s3_cm_principal.html) we made the account
'cloudera-scm/admin', so go ahead and put that username and whatever password you chose into this page.

From here you can just hit "Continue" until you're presented with a "Yes, I am ready to restart the cluster now"
checkbox. Check it, and (you guessed it!) hit "Continue" again. The wizard will now stop all the services and bring them
back up with Kerberos authentication activated.

That's it! You now have a Kerberos-secured CDH cluster!

This is cool in itself, but in my next post we'll get to the real heart of the matter: Installing and configuring Apache
Drill on this system so that we can perform the secure SQL queries that I advertized in the article title. Stay tuned!
