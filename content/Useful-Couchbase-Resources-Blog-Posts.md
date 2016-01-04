+++
date = "2013-08-06"
draft = false
title = "Useful Couchbase Resources & Blog Posts"
description = "This post contains a boatload of links and resources for your convenience. Bookmark and share!"
tags = ["couchbase","links","php","java","ruby","net","c"]
slug = "Useful-Couchbase-Resources-Blog-Posts"
+++

The following list is a convenient way to get access to lots of resources, blog posts and material that have been shared throughout the past months. I tried to separate them by area, but of course lots of them overlap to some extent. 

They are sorted by date (so you'll find the most recent ones on top) and include the author where possible. Be aware that some of the older articles may already be outdated  or not 100% accurate. 

I'll try to update this page as new articles get published, so it may pay off to come back here from time to time and check out the topmost ones. If you want to see your article included, post them in the comments!

## Starting Out
- [Official Couchbase Server 2.1 Manual](http://www.couchbase.com/docs/couchbase-manual-2.1.0/): The official manual and a good starting point for all kinds of general-purpose and architectural questions.
- [Couchbase Server 2.1 Developer Guide](http://www.couchbase.com/docs/couchbase-devguide-2.1.0/): The developer guide is particularly useful for developers that are new to document modeling and general app-related questions.
- [The Community Portal](http://www.couchbase.com/communities): Starting point for all SDK-related questions and links.
- 2013-04-24 [Top 10 things an Ops / Sys admin must know about Couchbase](http://blog.couchbase.com/top-10-things-ops-sys-admin-must-know-about-couchbase): Ten short rules that sys admins that maintain Couchbase clusters should keep in mind.
- 2012-07-06 [Couchbase 101 : Install, Store and Query Data (tgrall)](http://tugdualgrall.blogspot.co.at/2012/07/couchbase-101-install-store-and-query.html): A very basic introduction for people starting out, teaching the very basics in a hands-on fashion.

## Installation & Infrastructure
- [Official Installation Guide for 2.1](http://www.couchbase.com/docs/couchbase-manual-2.1.0/couchbase-getting-started.html): The official installation guide, read this before installing on production systems.
- 2013-08-05 [Running Couchbase 2.1.1 on SmartOS (trondn)](http://trondn.blogspot.no/2013/08/running-couchbase-211-on-smartos.html): Learn how to - step by step - compile and run a Couchbase cluster on SmartOS. This is for all the Solaris fans out there!
- 2013-07-22 [Deploying Couchbase with Chef (ketakigangal)](http://blog.couchbase.com/deploying-couchbase-chef): If you use Chef to automate your infrastructure, this blog post shows you how to integrate Couchbase with it.
- 2013-07-11 [Deploy your Node/Couchbase application to the cloud with Clever Cloud (tgrall)](http://tugdualgrall.blogspot.co.at/2013/07/deploy-your-nodecouchbase-application.html): You'll learn how to deploy your Node.JS app with Couchbase onto "clever cloud", a PaaS provider.
- 2013-06-27 [Nagios plugin to monitor Couchbase (Ebru Akagündüz)](http://www.ebruakagunduz.com/2013/06/nagios-plugin-to-monitor-couchbase.html?spref=tw): If you are using Nagios to monitor your infrastructure, this post shows you how to integrate Couchbase with a single plugin.
- 2013-05-31 [Create a Couchbase cluster in less than a minute with Ansible (tgrall)](http://tugdualgrall.blogspot.co.at/2013/05/create-couchbase-cluster-in-one-command.html): Create a Couchbase cluster automatically with Ansible, a systems automation framework like chef.
- 2013-05-27 [A Couchbase Cluster in Minutes with Vagrant and Puppet (daschl)](http://nitschinger.at/A-Couchbase-Cluster-in-Minutes-with-Vagrant-and-Puppet): If you want to setup a 4 node cluster with couchbase in minutes, this blog post shows you how to do it.

## Document Design & Data Import/Export
- 2013-07-18 [How to implement Document Versioning with Couchbase (tgrall)](http://tugdualgrall.blogspot.co.at/2013/07/how-to-implement-document-versioning.html): See how to implement document versioning by using a key-based approach. Uses the Java SDK, but can be adapted for all languages.
- 2013-07-08 [Denormalize the Datas for Great Good (John Connolly)](http://dev.theladders.com/2013/07/denormalize-the-datas-for-great-good): See how TheLadders benefited from denormalizing its dataset and using Couchbase, greatly reducing their response times.
- 2013-07-03 [SQL to NoSQL : Copy your data from MySQL to Couchbase (tgrall)](http://tugdualgrall.blogspot.co.at/2013/07/sql-to-nosql-importing-data-from-rdbms.html): A Java-based tool to import data from a SQL database into Couchbase.


## Views
- 2013-07-25 [Caching queries in Couchbase for high performance (Alexis Roos)](http://blog.couchbase.com/caching-queries-couchbase-high-performance): Learn how to cache view results for better performance.
- 2013-07-12 [Calculating average document size of documents stored in Couchbase. (Alexis Roos)](http://blog.couchbase.com/calculating-average-document-size-documents-stored-couchbase): With a simple map and a custom reduce function, one can easily calculate the average document size in the bucket.
- 2013-06-14 [Analyzing Binary Data in Couchbase (avsej)](http://avsej.net/2013/analyzing-binary-data-in-couchbase): Shows how to access binary (non-json) data from a View. Uses ruby to store the data, but can be adapted to any language.
- 2013-02-18 [How to get the latest document by date/time field (tgrall)](http://tugdualgrall.blogspot.co.at/2013/02/how-to-get-latest-document-by-datetime.html): A simple example on how to sort View-data based on a timestamp.
- 2013-02-13 [Introduction to Collated Views with Couchbase 2.0 (tgrall)](http://tugdualgrall.blogspot.co.at/2013/02/introduction-to-collated-views-with.html): Views can also be used to output "master/detail"-like scenarios. This post shows how.


## Java SDK & JVM
- [Getting Started & Download](http://www.couchbase.com/communities/java/getting-started): The official "Getting Started" page for the Java SDK. Introduction, Tutorial and Downloads.
- 2013-05-16 [Logging with the Couchbase Java Client (daschl)](http://nitschinger.at/Logging-with-the-Couchbase-Java-Client): An in-depth post about how to correctly configure logging for the Java SDK (and the underlying spymemcached library).
- 2013-04-17 [Couchbase Java SDK Internals (daschl)](http://nitschinger.at/Couchbase-Java-SDK-Internals): A very detailed post about the inner workings of the Java SDK. Recommended for advanced users who want to understand more of the internals.
- 2012-12-30 [Couchbase 101: Create views (MapReduce) from your Java application (tgrall)](http://tugdualgrall.blogspot.co.at/2012/12/couchbase-101-create-views-mapreduce.html): How to create views and design documents directly from the Java SDK.
- 2012-11-05 [Couchbase : Create a large dataset using Twitter and Java (tgrall)](http://tugdualgrall.blogspot.co.at/2012/11/couchbase-create-large-dataset-using.html): Feed data from Twitter directly into your Couchbase cluster through the Java SDK.
- 2012-04-26 [Accessing Couchbase from Scala (daschl)](http://nitschinger.at/Accessing-Couchbase-from-Scala): How to access the Couchbase Java SDK from the Scala programing language.



## .NET
- [Getting Started & Download](http://www.couchbase.com/communities/net/getting-started): The official "Getting Started" page for the .NET SDK. Introduction, Tutorial and Downloads.
- 2013-06-14 [Video: Code-First NoSQL with .NET and Couchbase (John Zablocki)](http://vimeo.com/68378224): A video where John Zablocki gives an introduction into NoSQL development and especially with Couchbase.
- 2013-03-06 [.NET Couchbase Client Instrumentation with ASP.NET and Glimpse (John Zablocki)](http://blog.couchbase.com/net-couchbase-client-instrumentation-aspnet-and-glimpse): See how to get your Couchbase server-side logging errors easily displayed in the browser console. Very helpful during development.
- 2013-02-01 [Moving No Schema up the Stack with C# and Dynamic Types (John Zablocki)](http://blog.couchbase.com/moving-no-schema-stack-c-and-dynamic-types): This blog post shows how to store schemaless data with both dictionaries and C#'s dynamic typing.
- 2013-01-04 [XDCR with ASP.NET and Nancy (John Zablocki)](http://blog.couchbase.com/xdcr-aspnet-and-nancy): Learn how to build an XDCR endpoint (like ElasticSearch integration) and read the data through our XDCR mechanisms.
- 2012-10-23 [Using C# Domain Objects to Define Couchbase Views (John Zablocki)](http://blog.couchbase.com/using-c-domain-objects-define-couchbase-views): How to automatically create DesignDocuments based of C# domain objects.
- 2012-10-05 [New Visual Studio Code Snippets for the .NET Couchbase Client Library (John Zablocki)](http://blog.couchbase.com/new-visual-studio-code-snippets-net-couchbase-client-library): A collection of helpful snippets in the day-to-day development process.
- 2012-09-20 [Strongly Typed Views with the .NET Client Library (John Zablocki)](http://blog.couchbase.com/strongly-typed-views-net-client-library): Learn how to map View responses directly onto domain objects.
- 2012-08-01 [Introducing the Couchbase ASP.NET OutputCache Provider (John Zablocki)](http://blog.couchbase.com/introducing-couchbase-aspnet-outputcache-provider): A short post on how to use Couchbase for easy caching in the ASP.NET environment.


## PHP
(Note: some of these posts are outdated in the way that currently the "way to go" when installing the PHP SDK is through PECL. See the the [Getting Started & Download](http://www.couchbase.com/communities/php/getting-started) guide for more information.)

- [Getting Started & Download](http://www.couchbase.com/communities/php/getting-started): The official "Getting Started" page for the PHP SDK. Introduction, Tutorial and Downloads.
- 2013-04-02  [Couchbase, PHP, XAMPP and Windows (trondn)](http://trondn.blogspot.co.at/2013/04/couchbase-php-xampp-and-windows.html): A short post on how to use the PHP SDK from Microsoft Windows.
- 2013-04-01 [Building Couchbase PHP driver on Ubuntu (trondn)](http://trondn.blogspot.co.at/2013/04/building-couchbase-php-driver-on-ubuntu.html): Learn how to build the Couchbase driver on Ubuntu and use it in a simple program.
- 2013-02-04 [Accessing Couchbase from PHP on your Mac! (trondn)](http://trondn.blogspot.co.at/2013/02/accessing-couchbase-from-php-on-your-mac.html) Building and running the SDK on your Mac is easy, learn how to do it in this post.
- 2012-11-01 [Building the PHP extension for Couchbase on Microsoft Windows! (trondn)](http://trondn.blogspot.co.at/2012/11/building-php-extension-for-couchbase-on.html): Learn how to compile the SDK on Windows using Visual Studio 2008.
- 2012-06-25 [How to store PHP sessions in Couchbase (daschl)](http://nitschinger.at/How-to-store-PHP-sessions-in-Couchbase): This post shows how to store PHP sessions in Couchbase using different mechanims (not only the official SDK).
- 2012-06-21 [Using Couchbase as a flexible session store (daschl)](http://nitschinger.at/Using-Couchbase-as-a-flexible-session-store): With Couchbase Server 2.0, JSON data and Views, its easy to run metrics over your sessions and identify user behavior. Learn how in this post.


## C & Go
- [Getting Started & Download with the C SDK](http://www.couchbase.com/communities/c/getting-started): The official "Getting Started" page for the C SDK. Introduction, Tutorial and Downloads.
- [Official Go Client Repository](https://github.com/couchbaselabs/go-couchbase): The Go repository with code and simple examples.


## Ruby
- [Getting Started & Download](http://www.couchbase.com/communities/ruby/getting-started): The official "Getting Started" page for the Ruby SDK. Introduction, Tutorial and Downloads.
- 2013-02-23 [Couchbase and Rails Talk (avsej)](http://avsej.net/links/2013/couchbase-and-rails/): A presentation by our lead Ruby SDK developer on how to integrate it with Ruby on Rails.
- 2013-02-11 [Using Couchbase Ruby Gem with EventMachine (avsej)](http://blog.couchbase.com/using-couchbase-ruby-gem-eventmachine): This post shows how to use the Ruby SDK together with the high performance EventMachine (custom protocols and performance for TCP/IP) gem.


## Node.JS
- 2013-03-06 [Easy application development with Couchbase, Angular and Node (tgrall)](http://tugdualgrall.blogspot.co.at/2013/03/easy-application-development-with.html): Storing Ideas and Votes in Couchbase with Angular and NodeJS.
- 2013-01-04 [Getting started with Couchbase and node.js on Windows (tgrall)](http://tugdualgrall.blogspot.co.at/2013/01/getting-started-with-couchbase-and.html): How to install and use the NodeJS Couchbase client library on Windows.
- 2012-11-13 [Building a chat application using Node.js and Couchbase (tgrall)](http://tugdualgrall.blogspot.co.at/2012/11/building-chat-application-using-nodejs.html): A nice chat application using Couchbase, NodeJS and socket.io for "real time" feeling.
- 2012-09-24 [Create a Simple Node.js and Couchbase application... on OS X (tgrall)](http://tugdualgrall.blogspot.co.at/2012/09/create-simple-nodejs-and-couchbase.html) A simple (but maybe outdated) tutorial on how to use the NodeJS driver from OSX.

## Python
- [Getting Started & Download](http://www.couchbase.com/communities/python/getting-started): The official "Getting Started" page for the Python SDK. Introduction, Tutorial and Downloads.
- 2013-06-21 [Python Extension Windows Binaries (mnunberg)](http://mnunberg.github.io/2013/python-extension-windows-binaries): A tale by our maintainer of the Python SDK on how to upload Windows binaries to PyPi.
- 2013-05-30 [What's up with the Python Couchbase SDK (volker)](http://blog.couchbase.com/whats-python-couchbase-sdk): A rundown of the latest changes on the Python SDK.


## Ecosystem
- 2013-07-31 [Couchbase: Cluster Setup + ORM Secondary Cache Introduction (Brad Wood)](http://www.ortussolutions.com/blog/couchbase-cluster-setup-orm-secondary-cache-introduction): Using ColdFusion and looking for a secondary cache implementation? Look no further.
- 2013-07-22 [Real-time visitor analysis with Couchbase, Elasticsearch and Kibana (Jeroen Reijn)](http://blog.jeroenreijn.com/2013/07/visitor-analysis-with-couchbase-elasticsearch.html): A great post on a customer's use case for real-time visitor analysis.


## Troubleshooting
- [Official troubleshooting Guide](http://www.couchbase.com/docs/couchbase-manual-2.1.0/couchbase-troubleshooting.html): The best place to start if something goes wrong on the server and you don't know why.
- 2012-12-26 [What to do if your Couchbase Server does not start? (tgrall)](http://tugdualgrall.blogspot.co.at/2012/12/what-to-do-if-your-couchbase-server.html): If you have troubles getting older Couchbase Server 2.0 versions to run on Windows, this post is for you.


## Fun Stuff
- [Example code for lots of SDKs and Languages](https://github.com/couchbaselabs/DeveloperDay): Our DeveloperDay material with hands-on code examples to try and learn.
- 2013-06-18 [Fun with Couchbase Views and MessagePack (daschl)](http://nitschinger.at/Fun-with-Couchbase-Views-and-Message-Pack): While JSON is tried and true, with a little twiggling and some fun you can get Couchbase Views to speak MessagePack!
- 2013-04-29 [Screencast : Fun with Couchbase, MapReduce and Twitter (tgrall)](http://tugdualgrall.blogspot.co.at/2013/04/screencast-fun-with-couchbase-mapreduce.html): A Screencast on importing Twitter data into Couchbase and analyzing it on the fly through Views.

Last Updated: 2013-08-06 (daschl)
