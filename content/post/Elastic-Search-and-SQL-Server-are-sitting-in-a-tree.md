+++
date = "2012-08-20"
draft = false
title = "ElasticSearch and SQL Server are sitting in a tree..."
description = "Want to know how to integrate SQL Server and ElasticSearch? Read on."
tags = ["elasticsearch","sql server","jdbc"]
slug = "Elastic-Search-and-SQL-Server-are-sitting-in-a-tree"
disqus_identifier = "bd93492f22cc9e544007a22928c8e2dc49cf788c53c23be2b51cbf84ecec98c55ed01e26bbd4d20ebaf23e47203fbe9e2e1fdf7ef06143c9fd3a675cddc7f5c0"
+++

## Motivation
With modern NoSQL datastores on the rise, classical relational databases with their rigid data model get challenged every day. Nonetheless, they still own the market and therefore every developer needs to have solid skills in working with them in a reliable and performant way.

While relational databases have lots of use cases, there are areas where different technologies are a much better fit. One of them is flexible and complex real-time searching. Most of you have probably heard of [Apache Lucene](http://lucene.apache.org/core/), a very mature and feature-rich search engine that has been around for a long time. Lucene is great, but it also has a few shortcomings. [ElasticSearch](http://www.elasticsearch.org/) builds on the trusted and performant base of Lucene and adds lots of compelling features (JSON documents, RESTful API or automatic rebalancing to name just a few). Also, the team behind ES joined forces recently and is now offering [commercial support](http://www.elasticsearch.com/) as well. 

This post shows you how to integrate Microsoft SQL Server with ElasticSearch. We'll automatically pull data from SQL Server into ElasticSearch and make sure the data is up-to-date and ready to query at any time. The solution provided here came out of a real-world use-case and is deployed to production successfully. The use case I had is maybe the same for you: index a specific set of documents that should be searchable in near-real time from a website. In my case these documents are trouble tickets, but ES doesn't care what you index at all.

I don't expect you to have lots of knowledge with ElasticSearch, but it helps if you've played with it already. Also, very basic SQL knowledge is needed since we use it to pull out data from SQL Server.

Also note that since we use JDBC to fetch the data, it is very easy to switch over to a MySQL, PostgreSQL or Oracle JDBC driver in no time and reuse exactly the same code.

First, we're going to install ElasticSearch and all the needed plugins and libraries. Then, we make sure everything is up and running and create a `river` (more on that later) that automatically pulls the data out of our SQL Server.

## Preparations
The first thing we need to do is download and extract ElasticSearch. Since it is written in Java, we also need to have a JDK (1.6) installed and in our `PATH`. The code provided here is from my Ubuntu Linux machine, but this also works on Windows.

Download the latest archive (zip or tgz) from [here](http://www.elasticsearch.org/download/). There are debian packages available, but I'm using the self-contained archive here because this approach works on Windows as well and I want this post to target a broader audience.

	michael@daschl:~$ echo $JAVA_HOME
	/usr/lib/jvm/java-6-openjdk-i386/
	michael@daschl:~/Downloads$ tar -xvzpf elasticsearch-0.19.8.tar.gz 
	...
	michael@daschl:~/Downloads$ cd elasticsearch-0.19.8/

That's it, your search engine is already prepared to do the magic. We'll start it in the foreground just to see what's going on on the console (if you leave the `-f` switch away, then ES starts as a daemon. You can still find the same output inside the `logs` directory):

	michael@daschl:~/Downloads/elasticsearch-0.19.8$ bin/elasticsearch -f
	[2012-08-17 19:42:47,719][INFO ][node                     ] [Jackson Arvad] {0.19.8}[3236]: initializing ...
	[2012-08-17 19:42:47,725][INFO ][plugins                  ] [Jackson Arvad] loaded [], sites []
	[2012-08-17 19:42:49,698][INFO ][node                     ] [Jackson Arvad] {0.19.8}[3236]: initialized
	[2012-08-17 19:42:49,698][INFO ][node                     ] [Jackson Arvad] {0.19.8}[3236]: starting ...
	[2012-08-17 19:42:49,777][INFO ][transport                ] [Jackson Arvad] bound_address {inet[/0:0:0:0:0:0:0:0:9300]}, publish_address {inet[/192.168.1.101:9300]}
	[2012-08-17 19:42:52,809][INFO ][cluster.service          ] [Jackson Arvad] new_master [Jackson Arvad][TAnCKhf0RymniOxTrT8nww][inet[/192.168.1.101:9300]], reason: zen-disco-join (elected_as_master)
	[2012-08-17 19:42:52,888][INFO ][discovery                ] [Jackson Arvad] elasticsearch/TAnCKhf0RymniOxTrT8nww
	[2012-08-17 19:42:52,919][INFO ][http                     ] [Jackson Arvad] bound_address {inet[/0:0:0:0:0:0:0:0:9200]}, publish_address {inet[/192.168.1.101:9200]}
	[2012-08-17 19:42:52,919][INFO ][node                     ] [Jackson Arvad] {0.19.8}[3236]: started
	[2012-08-17 19:42:53,012][INFO ][gateway                  ] [Jackson Arvad] recovered [0] indices into cluster_state


If you point your browser to `localhost:9200`, you should see a JSON response indicating that our instance is running:

	{
	  "ok" : true,
	  "status" : 200,
	  "name" : "<SOME COOL RANDOM NAME>",
	  "version" : {
	    "number" : "0.19.8",
	    "snapshot_build" : false
	  },
	  "tagline" : "You Know, for Search"
	}

Okay, that wasn't rocket science but at least we are prepared to install everything else. We need two more things to get everything going: the JDBC river itself and the appropriate SQL Server JDBC driver.

The river is available on [GitHub](https://github.com/jprante/elasticsearch-river-jdbc), but the easiest way is to install it through the `plugin` command, which is bundled with our ES installation:

	michael@daschl:~/Downloads/elasticsearch-0.19.8$ bin/plugin -install jprante/elasticsearch-river-jdbc/1.3.1
	-> Installing jprante/elasticsearch-river-jdbc/1.3.1...
	Trying https://github.com/downloads/jprante/elasticsearch-river-jdbc/elasticsearch-river-jdbc-1.3.1.zip...
	Downloading .............................DONE
	Installed river-jdbc

You can also [download](https://github.com/jprante/elasticsearch-river-jdbc/downloads) the archive and extract it into the `plugins` directory, but you need to make sure the directory name is `river-jdbc`.

Assuming that no SQL Server JDBC driver is in your path, you have two choices available: you can either use the free [jTDS](http://jtds.sourceforge.net/) driver or the proprietary [driver from Microsoft](http://msdn.microsoft.com/en-us/sqlserver/aa937724.aspx). We first tried the jTDS driver, but had mayor problems while trying to index CLOB columns. We then switched over to the Microsoft driver and everything worked as expected.

If you're using an old SQL Server (like version 8/2000) and the 1.6 JDK, you have to use the JDBC Driver version 3, otherwise you're good to go with the current version 4. [Download](http://www.microsoft.com/en-us/download/details.aspx?displaylang=en&id=11774), extract and copy the `sqljdbc4.jar` into the `lib` directory. It doesn't have to be the `lib` directory by the way, just make sure it's in your classpatch so that ES is able to find it during runtime.

We've now installed all mandatory dependencies, but there is one more I'd like to recommend: [elasticsearch-head](http://mobz.github.com/elasticsearch-head/). It's a very convenient frontend to work with your cluster, since it allows you to manipulate indexes and data through a pure HTML5-based interface. You can install it either as a plugin or just download the archive and open up the `index.html` file. Since we've already used the `plugin` command, let's do it again:

	michael@daschl:~/Downloads/elasticsearch-0.19.8$ bin/plugin -install mobz/elasticsearch-head
	-> Installing mobz/elasticsearch-head...
	Trying https://github.com/downloads/mobz/elasticsearch-head/elasticsearch-head-0.19.8.zip...
	Trying https://github.com/mobz/elasticsearch-head/zipball/v0.19.8...
	Trying https://github.com/mobz/elasticsearch-head/zipball/master...
	Downloading ....................................DONE
	Identified as a _site plugin, moving to _site structure ...
	Installed head

Now, after you've started ES again, head over to `http://localhost:9200/_plugin/head/` and enjoy the awesomeness!

## Configuring the River
In ES, you command everything through the REST API. In practice, this means shuffling JSON documents around. I recommend you to read into the [ES documentation](http://www.elasticsearch.org/guide/) first to get a feeling how everything works in general. Then, head over to the [river](http://www.elasticsearch.org/guide/reference/river/) documentation to learn more about them.

Basically, a river is a library to pull data from an external source into the cluster. There are lots of other rivers available, like the one for [CouchDB](http://www.elasticsearch.org/guide/reference/river/couchdb.html) or for [twitter](http://www.elasticsearch.org/guide/reference/river/twitter.html).

To create the river, we'll send the following PUT request to `localhost:9200/_river/my_jdbc_river/_meta`:

	{
		"type":"jdbc",
		"jdbc": {
			"driver":"com.microsoft.sqlserver.jdbc.SQLServerDriver", 
			"url":"jdbc:sqlserver://127.0.0.1:1433;databaseName=myDatabase",
			"user":"username","password":"password",
			"sql":"select * from tablename",
			"poll":"30s"
		},
		"index": {
			"index":"indexname",
			"type":"typename",
			"bulk_size":500
		}
	}

The URL uniquely identifies our river since we are free to add more than one. Let's break the JSON document down a bit:
	
	"type":"jdbc",

The `type` identifies the river plugin, since there are lots of different rivers available. For example, the CouchDb river is identified by a `couchdb` type.

	"jdbc": {
		"driver":"com.microsoft.sqlserver.jdbc.SQLServerDriver", 
		"url":"jdbc:sqlserver://127.0.0.1:1433;databaseName=myDatabase",
		"user":"username",
		"password":"password",
		"sql":"select * from tablename",
		"poll":"30s"
	},

This block provides the main configuration params for the river. The `driver` identifies the JDBC driver that we want to use, so if you're going to use jTDS you should use `net.sourceforge.jtds.jdbc.Driver` instead. The `url` represents the JDBC connection string, which also contains the database name at the end. These strings also vary from driver to driver. Since you always need to provide a username and a password, the river provides `user` and `password` fields accordingly.

Now it starts to get interesting. The `sql` key is the statement that the river executes against the database to fetch the rows. If you only want to index a subset of your columns, just use the explicit column list (like `SELECT foo, bar FROM ...`). You are free to do everything in here that is valid SQL, so if you like you can add joins, conditions, aliases and ordering.

The `poll` key defines the time interval between the data is pulled from the database. In this configuration, all rows are fetched every 30 seconds and (re)indexed in ES.

	"index": {
		"index":"indexname",
		"type":"typename",
		"bulk_size":500
	}

The `index` configuration block allows you to customize your index metadata. If this concept is not familiar to you, think of the `indexname` as some kind of database and `typename` as the table name. You also need this information to locate your indexed data later. So, if you want to index tweets and users from twitter, a good approach would be to name the index `twitter` and define two types - one for `users` and one for `tweets`.

The bulk size defines how many records are hold back until they're indexed in one batch. I don't know what bulk size is reasonable for your use-case, but you may want to start at `100` and test higher values if they provide performance increases.

The easiest way to send the request is through the "Any Request" functionality in es-head, but you can also do it through CURL or even your favorite programming language. If everything went well, you should receive a response like this:

	{

	    ok: true
	    _index: _river
	    _type: my_jdbc_river
	    _id: _meta
	    _version: 1
	}

The `_version` increases every time you issue a new PUT request to update the river. The easiest way to start the indexing is to restart the server, since the river fires off immediately after restart (you can use the `kill -15` command to stop it in a controlled way). Your output should look roughly like this, indicating that the data gets pulled and indexed (you'll see nearly the same output after every `poll` interval):

	michael@daschl$ sudo bin/elasticsearch
	michael@daschl$ tail -f logs/elasticsearch
	....
	[2012-08-20 08:14:40,407][INFO ][river.jdbc               ] [Perfection] [jdbc][my_jdbc_river] got 30561 rows for version 2710, digest = RFEDCNwLdQabcedfegggRszqzy1MeroiGEejB88=
	[2012-08-20 08:14:40,407][INFO ][river.jdbc               ] [Perfection] [jdbc][my_jdbc_river] submitting new bulk request (61 docs, 3 requests currently active)
	[2012-08-20 08:14:40,445][INFO ][river.jdbc               ] [Perfection] [jdbc][my_jdbc_river] waiting for 3 active bulk requests
	[2012-08-20 08:14:40,511][INFO ][river.jdbc               ] [Perfection] [jdbc][my_jdbc_river] bulk request success (104 millis, 61 docs, total of 23061 docs)
	[2012-08-20 08:14:40,511][INFO ][river.jdbc               ] [Perfection] [jdbc][my_jdbc_river] waiting for 2 active bulk requests
	[2012-08-20 08:14:40,513][INFO ][river.jdbc               ] [Perfection] [jdbc][my_jdbc_river] bulk request success (590 millis, 500 docs, total of 23561 docs)
	[2012-08-20 08:14:40,513][INFO ][river.jdbc               ] [Perfection] [jdbc][my_jdbc_river] waiting for 1 active bulk requests
	[2012-08-20 08:14:40,622][INFO ][river.jdbc               ] [Perfection] [jdbc][my_jdbc_river] bulk request success (274 millis, 500 docs, total of 24061 docs)
	[2012-08-20 08:14:40,624][INFO ][river.jdbc               ] [Perfection] [jdbc][my_jdbc_river] next run, waiting 30s, URL [jdbc:sqlserver://127.0.0.1:1433;databaseName=myDatabase] driver [com.microsoft.sqlserver.jdbc.SQLServerDriver] sql [select * from tablename] river table [false]

The last log message indicates that the data has been indexed and ES patiently waits for the specified time interval before the new run starts. We can now fetch some documents, either through es-head or via curl. The following request returns the first 50 matches from all indexed documents under your `indexname`:

	POST to http://127.0.0.1:9200/indexname/_search
	{
	  "query": {
	    "bool": {
	      "must": [
	        {
	          "match_all": {}
	        }
	      ]
	    }
	  },
	  "from": 0,
	  "size": 50
	}

The JDBC river provides a bunch of advanced options that you also want to look at, especially [how to manage updates](https://github.com/jprante/elasticsearch-river-jdbc#managing-updates-with-the-river-table). As far as I know, there is currently no support for incremental updates, but the author plans to implement it in one of the next versions.

## Wrapping Up
After reading this post, you should be able to install, configure and query ElasticSearch together with a JDBC river. We used SQL Server here, but the same principles apply to all other relational databases with a JDBC driver as well.

The JDBC river provides you with an easy way to pull data from relational databases into your ES cluster, but it has some shortcomings that you need to be aware of. The most problematic one is that it doesn't support incremental updates, which is why you may need to wrap some more logic around your data stream. Aside from that, our production deployment is running pretty stable and there haven't been any other major problems until now.

As always, feel free to post your feedback, corrections or improvement suggestions below!
