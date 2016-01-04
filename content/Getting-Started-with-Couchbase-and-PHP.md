+++
date = "2012-03-12"
draft = false
title = "Getting Started with Couchbase and PHP"
description = "This post gets you started with Couchbase and its PHP SDK."
tags = ["couchbase","php","sdk"]
slug = "Getting-Started-with-Couchbase-and-PHP"
+++

[Couchbase](http://www.couchbase.com/) is a simple, fast and elastic document-oriented database. It is the product of a merge between the companies behind Memcached (Membase) and CouchDB. The current version is 1.8 and the 2.0 version is already in a developer preview release. If you haven't heard much about Couchbase yet, start at the [Couchbase Webinar Series](http://www.couchbase.com/webinar/couchbase-server-2.0-overview), which will get you up to speed. The full manual is available [here](http://www.couchbase.com/docs/couchbase-manual-2.0/index.html).

As there were a lot of merges, renamings and releases, it was pretty hard to follow up with the current/best database version and SDK to use for your project. Now as the dust has settled a bit, here's what I've come up with:

Couchbase 2.0 will be the next major version and is already pretty stable, so I'll jump straight onto it and skip 1.8. When it comes to PHP SDKs, you should either use the more feature-complete [php-couchbase](https://github.com/couchbaselabs/php-couchbase) SDK (which is based on pecl/memcached) or work the brand new [php-ext-couchbase](https://github.com/couchbaselabs/php-ext-couchbase) extension. Note that it doesn't support working with views currently, but according to [janl](https://twitter.com/#!/janl) this will be available in a few days. He also mentioned that this extension is the 
future way of interacting with Couchbase, so it is definitely worth a look.

The rest of this post guides you through the installation process of both Couchbase and the PHP SDKs.

Installing Couchbase
--------------------
The easiest way to install Couchbase is through a precompiled binary for Ubuntu, RedHat, Windows or Mac OS X. You can find all of them 
[here](http://www.couchbase.com/download). The installation process is really straightforward: you first need to install the binary and 
then configure Couchbase from a web interface. I recommend you to watch the webinar starting from minute 7 [here](http://www.couchbase.com/webinar/couchbase-server-2.0-overview), where Perry Krug describes the necessary steps in detail. For the purpose of this post, you can just stick with the default settings.

	$ sudo dpkg -i couchbase-server-community_x86_2.0.0-dev-preview-3.deb

After the installation, you should have a Couchbase instance listening on `127.0.0.1:8091`.

Installing php-ext-couchbase
----------------------------
The new PHP SDK ships as a PHP extension, so we need to compile it. Make sure you have both the build tools (like `build-essentials`) and the PHP development packages (`php5-dev php-pear`) installed. You also need to install `libssl0.9.8`. On Ubuntu it looks like this:

	$ sudo aptitude install libssl0.9.8 php5-dev php-pear

The extension also depends on the `libcouchbase` library, which can be installed through the Couchbase repository (for more information on this, look [here](http://www.couchbase.com/develop/c/current)).

Add this to your `/etc/apt/sources.list`:

	# Ubuntu 11.10 Oneiric Ocelot (Debian unstable)
	deb http://packages.couchbase.com/ubuntu oneiric oneiric/main

Then, import the key, update your repositories and install the package:

	$ wget -O- http://packages.couchbase.com/ubuntu/couchbase.key | sudo apt-key add -
	$ sudo aptitude update && sudo aptitude install libcouchbase-dev

Now, we need to [download](https://github.com/couchbaselabs/php-ext-couchbase/zipball/master) the extension and build it.

	$ unzip couchbaselabs-php-ext-couchbase-2680b64.zip
	$ cd couchbaselabs-php-ext-couchbase-2680b64
	
	$ phpize
	$ ./configure
	$ make
	$ sudo make install
	  Installing shared extensions:     /usr/lib/php5/20090626+lfs/

The last thing we need to do is create a new `couchbase.ini` file in `/etc/php5/conf.d` with the following content:

	extension=couchbase.so

Now we can see that the couchbase extension is installed correctly:

	$ php -m | grep couch
	couchbase

Of course you'll need to restart your webserver or your fastcgi processes too.

Installing php-couchbase
------------------------
If you want to work with the current stable SDK, you need to download the code from the [php-couchbase](https://github.com/couchbaselabs/php-couchbase) repository. This SDK extends the `memcached` extension, which you can install directly through apt:

	$ sudo aptitude install php5-memcached

Documentation for the PHP 1.1 SDK can be found [here](http://www.couchbase.com/docs/couchbase-sdk-php-1.1/getting-started.html).

Note that for the rest of this post I'll use the API provided by the `php-ext-couchbase` extension, but the one for `php-couchbase` shouldn't be much different.

Basic Usage
-----------
Historically, communication with CouchDB was done over pure HTTP while memcached provided a custom protocol. Couchbase uses both methods, depending on the task you want to accomplish. Create, update and delete actions are done through the `memcached` protocol. If you want to read data, you can do that either based on the key (through the memcached protocol) or on a view (through HTTP). Thankfully, the SDK abstracts this away from you, but its always good if you know whats going on under the hood.

You can access the API through an object-oriented interface or directly through their exported functions. I couldn't find any online documentation for now, but here's the function list from the source-code:

	/* {{{ couchbase_functions[]
	*/
	static zend_function_entry couchbase_functions[] = {
		PHP_FE(couchbase_connect, arginfo_connect)
		PHP_FE(couchbase_add, arginfo_add)
		PHP_FE(couchbase_set, arginfo_set)
		PHP_FE(couchbase_set_multi, arginfo_set_multi)
		PHP_FE(couchbase_replace, arginfo_replace)
		PHP_FE(couchbase_prepend, arginfo_prepend)
		PHP_FE(couchbase_append, arginfo_append)
		PHP_FE(couchbase_cas, arginfo_cas)
		PHP_FE(couchbase_get, arginfo_get)
		PHP_FE(couchbase_get_multi, arginfo_get_multi)
		PHP_FE(couchbase_get_delayed, arginfo_get_delayed)
		PHP_FE(couchbase_fetch, arginfo_fetch)
		PHP_FE(couchbase_fetch_all, arginfo_fetch_all)
		PHP_FE(couchbase_increment, arginfo_increment)
		PHP_FE(couchbase_decrement, arginfo_decrement)
		PHP_FE(couchbase_get_stats, arginfo_get_stats)
		PHP_FE(couchbase_delete, arginfo_delete)
		PHP_FE(couchbase_flush, arginfo_flush)
		PHP_FE(couchbase_get_result_code, arginfo_result_code)
		PHP_FE(couchbase_set_option, arginfo_set_option)
		PHP_FE(couchbase_get_option, arginfo_get_option)
		PHP_FE(couchbase_version, arginfo_version)
		{NULL, NULL, NULL}
	};

The [test cases](https://github.com/couchbaselabs/php-ext-couchbase/tree/master/tests) also provide vital information on how to use the API.

Let's create 10 JSON documents and store them in Couchbase:

	<?php

	$connection = new Couchbase("127.0.0.1:8091");

	for($i=0;$i<10;$i++) {
		$doc = array(
			'title' => "This is Post $i"
		);
		$connection->set("post_$i", json_encode($doc));
	}

	?>

You can view your stored documents instantly through the GUI (http://localhost:8091/index.html#sec=documents&bucketName=default). Now we can retrieve all documents again by their keys:

	<?php

	$connection = new Couchbase("127.0.0.1:8091");

	echo "<ul>";
	for($i=0;$i<10;$i++) {
		$post = json_decode($connection->get("post_$i"));
		echo "<li>" . $post->title .  "</li>";
	}
	echo "</ul>";

	?>

Note that we need to encode/decode our JSON documents by hand, because in Couchbase you can store pretty much what you want in a document.

There are lots of other things that can be explored, but for now you should have a Couchbase instance up and running and you can start to explore it on your own!

'''Edit 1:'''
Jan [pointed out](https://twitter.com/#!/janl/status/179166150205259776) that they don't recommend php-couchbase at this point as the extension will gain all functionality soon. Thanks!