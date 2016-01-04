+++
date = "2012-06-25"
draft = false
title = "How to store PHP sessions in Couchbase"
description = "In this post I'll show you how to store your sessions in Couchbase through three different approaches."
tags = ["php","couchbase","sessions"]
slug = "How-to-store-PHP-sessions-in-Couchbase"
+++

My [last post](http://nitschinger.at/Using-Couchbase-as-a-flexible-session-store) about storing sessions in Couchbase was more of a general overview on what's possible. This post builds on the foundations and principles discussed and shows you various ways to store PHP sessions in Couchbase. We start out simple by using the built-in `session` ini -directives, then head over to the brand new [SessionHandlerInterface](http://php.net/manual/en/class.sessionhandlerinterface.php) introduced in PHP 5.4 and finally implement it completely on our own. We'll see how both the complexity and flexibility increase during each of our approaches. In the end it's up to you which method seems appropriate for your use-case.

## session_handler and memcached
This approach has one big advantage: it's dead easy to setup and use. Let's tackle the good parts first and then discuss the weaknesses of this approach.

We are using the protocol-compatible nature of Couchbase and let the built-in `memcached` session handler do the job for us. So the first thing we need to do (apart from installing Couchbase) is install the `php5-memcached` extension. If you're on a debian-based system it's as easy as running

	$ sudo aptitude install php5-memcached

If you want to make sure that the extension is correctly installed, check the module on the command line:

	$ php -m | grep mem
	memcached

Now we need to tell PHP to use the `memcached` session handler and where it should save the data. To do this, we need to modify the `php.ini` file accordingly. I'm using [Apache](http://httpd.apache.org/) on [Ubuntu](http://www.ubuntu.com/), so my `php.ini` is located under `/etc/php5/apache2/php.ini`. Find and replace the following settings with their new values:

	[Session]
	session.save_handler = memcached
	session.save_path = "localhost:11211"

Of course, you want to replace `localhost` with a different address if Couchbase is not running on your local machine. Couchbase uses `11211` as its [data port](http://www.couchbase.com/docs/couchbase-manual-2.0/couchbase-network-ports.html), just like memcached. Now we have everything in place, restart the server to get it up and running:

	$ sudo service apache2 restart

The infrastructure is in place, so we can now start working with the session store through the `$_SESSION` superglobal. Here is a short script that shows how you can work with it:

	<?php

	// Start the session.
	session_start();

	// Dump the session to the screen.
	var_dump($_SESSION);

	// Define some data to store.
	class User { public $firstname = "hoo"; }
	$user = new User();

	// Store it in the session object.
	$_SESSION['user'] = $user;
	$_SESSION['randomStuff'] = array(1 => 'blue', 'couchbase' => 'rocks');

	?>

This is not rocket science and exactly the way you'd handle sessions through the interface provided by PHP. Check out the [documentation](http://php.net/manual/en/reserved.variables.session.php) if you haven't used it yet.

So, how does PHP store the data in Couchbase? The first thing to note is that all your session data is stored in the `default` bucket and uses a binary format (like how it would be stored in memcached). If you inspect the output from our example above, you can see that the user object is actually stored as an object so PHP uses the built-in serializer (if not configured differently) to store the session data. If you look in the bucket, you can see a key like this:

	memc.sess.key.a6sgl1oldchknlsj8mc3c37b11

The first part is the prefix and the cryptic string at the end is the PHP session identifier which is carried along by the client through a cookie. If you fire up firebug or chrome and inspect the headers, you can see the corresponding `PHPSESSID`:

	Cookie:PHPSESSID=a6sgl1oldchknlsj8mc3c37b11;

The stored document looks like this:

	{
	"_id": "memc.sess.key.a6sgl1oldchknlsj8mc3c37b11",
	"_rev": "10-000003794c342b870000009800000000",
	"$flags": 0,
	"$expiration": 0,
	"$att_reason": "invalid_json"
	}

Since the data is not stored in JSON format, you can't inspect it from the Couchbase GUI directly. So why did I include it here anyway? Look at the `$expiration` key. It is important to note that we don't specify an expiration time here, because this is handled through PHP. So if you want to change the expiration time you have to change the appropriate setting using the `session.cache_expire` directive. This is different to the approach we'll cover at the end of the article where we implement everything on our own and don't rely on those built-in features.

This approach is very simple and may suit your needs perfectly, but it comes with some limitations that you need to keep in mind.

Since the data is not stored in JSON format, you can't use the awesome map/reduce querying mechanisms that Couchbase 2.0 provides. Also, you can't change the bucket or key-names. Another problem is that you can't make use of the automatic, dynamic reconfiguration mechanisms provided by Couchbase. So if the server at the ip-address goes down, PHP won't be able to store sessions in the cluster anymore. Couchbase provides an optional component called [Moxi](http://www.couchbase.com/docs/moxi-manual-1.8/moxi-introduction.html) that acts as a proxy between your client and the couchbase cluster and is able to handle such situations. So if you're not able to use the approaches shown in the rest of this post you definitely need to take a look at it!

## More flexibility through the SessionHandlerInterface
PHP 5.4 provides a brand new interface called [SessionHandlerInterface](http://www.php.net/manual/en/class.sessionhandlerinterface.php) through which we can implement our own session handler. This way, we can use the familiar and built-in `$_SESSION` superglobal and make use of the advanced Couchbase functionality.

We need to implement the following interface:

	SessionHandlerInterface {
		abstract public bool close ( void )
		abstract public bool destroy ( string $session_id )
		abstract public bool gc ( string $maxlifetime )
		abstract public bool open ( string $save_path , string $session_id )
		abstract public string read ( string $session_id )
		abstract public bool write ( string $session_id , string $session_data )
	}

The following implementation is not meant to be used 1:1 in production, since I haven't tested every aspect of it. Nevertheless it should give you an idea on how such a implementation may look like. Note that you need to have the `php-ext-couchbase` extension installed. If you are not sure on how to do this, see my post about [Getting Started with Couchbase and PHP](http://nitschinger.at/Getting-Started-with-Couchbase-and-PHP). You can also find a github gist [here](https://gist.github.com/2986942).


	<?php

	/**
	* A reference implementation of a custom Couchbase session handler.
	*/
	class CouchbaseSessionHandler implements SessionHandlerInterface {

		/**
		* Holds the Couchbase connection.
		*/
		protected $_connection = null;

		/**
		* The Couchbase host and port.
		*/
		protected $_host = null;

		/**
		* The Couchbase bucket name.
		*/
		protected $_bucket = null;

		/**
		* The prefix to be used in Couchbase keynames.
		*/
		protected $_keyPrefix = 'session:';

		/**
		* Define a expiration time of 10 minutes.
		*/
		protected $_expire = 600;

		/**
		* Set the default configuration params on init.
		*/
		public function __construct($host = '127.0.0.1:8091', $bucket = 'default') {
			$this->_host = $host;
			$this->_bucket = $bucket;
		}

		/**
		* Open the connection to Couchbase (called by PHP on `session_start()`)
		*/
		public function open($savePath, $sessionName) {
			$this->_connection = new Couchbase($this->_host, $this->_bucket);
			return $this->_connection ? true : false;
		}

		/**
		* Close the connection. Called by PHP when the script ends.
		*/
		public function close() {
			unset($this->_connection);
			return true;
		}

		/**
		* Read data from the session.
		*/
		public function read($sessionId) {
			$key = $this->_keyPrefix . $sessionId;
			$result = $this->_connection->get($key);

			return $result ?: null;
		}

		/**
		* Write data to the session.
		*/
		public function write($sessionId, $sessionData) {
			$key = $this->_keyPrefix . $sessionId;
			if(empty($sessionData)) {
				return false;
			}

			$result = $this->_connection->set($key, $sessionData, $this->_expire);
			return $result ? true : false;
		}

		/**
		* Delete data from the session.
		*/
		public function destroy($sessionId) {
			$key = $this->_keyPrefix . $sessionId;
			$result = $this->_connection->delete($key);

			return $result ? true : false;
		}

		/**
		* Run the garbage collection.
		*/
		public function gc($maxLifetime) {
			return true;
		}

	}

	?>

It's not that complex as it may look like in the first place. At the top of the class we're defining and initializing some variables that hold our configuration. We don't have to do this, but this way we're getting a bit more flexibility and can pass custom settings through the constructor.

The `open` method takes two arguments (according to the interface definition), one for the `savePath` and one for the `sessionName`. These arguments correspond to the `ini`-settings, but we ignore them and make use of our configuration variables passed through the constructor. Inside the `open` method we initialize the connection to Couchbase and store the connection resource in a protected variable for later use.

The `close` method is called when the PHP-script ends. We just unset the resource, since there is nothing more to do in our case. If you use a file storage engine you'd want to close the file-handle here.

From now on, we always get the `sessionId` passed as an argument. The `read` method takes it and adds zhr `keyPrefix` we'ved defined to the beginning of the string. It then tries to read the record from Couchbase and returns it when it's found. Note that we return `null` here, but since `$_SESSION` is an array, you'll always work with (empty) arrays on the caller-side.

The `write` method gets the payload as an additional argument since we need to store it somewhere. Note that the data is already serialized, so don't try to use `json_encode` or something like this before storing it in Couchbase (that's what I did and then wondered what the hell is wrong unless someone on IRC pointed out that you can't control the serialization mechanism on your own here). As a result, the `read` method expects the serialized data as a string and fails silently if you return something else.

The `destroy` method should not contain any surprises, we just delete the document based on the key if it exists. The `gc` method is not used here, because Couchbase handles the garbage collection for us. If you store your data in files, you want to clean up old sessions here.

Now we can use our handler and do some work with it:

	<?php
	$handler = new CouchbaseSessionHandler();

	session_set_save_handler($handler, true);
	session_start();

	var_dump($_SESSION);

	$_SESSION['foo'] = null;
	$_SESSION['data'] = array(
		'couchbase' => 'rocks'
	);
	?>

On the first load, the `$_SESSION` variable is of course empty. When you reload the page, you should see output like this:

	array(2) { 'foo' => NULL 'data' => array(1) { 'couchbase' => string(5) "rocks" } }

In the Couchbase GUI, we can see that a key has been added to the `default` bucket named something like this:

	session:a6sgl1oldchknlsj8mc3c37b11

Couchbase provides a handy tool on the command line to inspect our documents - it's called `cbc`. We can use it to fetch the contents of our key and peek inside:

	$ cbc cat session:a6sgl1oldchknlsj8mc3c37b11
	"session:a6sgl1oldchknlsj8mc3c37b11" Size 45 Flags 0x0 CAS 0x9d394b19130a0000
	foo|N;data|a:1:{s:9:"couchbase";s:5:"rocks";}

You can see that the PHP session handler still stores the data in a serialized way (you can also `var_dump()` the content of the `$sessionData` variable handed over to our `write()` method to see the serialized data string).

This approach is a huge leap forward because it let's us shape the interaction with Couchbase the way we want it to (and we can leverage all commands from the Couchbase PHP-SDK instead of just using a subset provided by the memcached extension). Also, we use the expiration mechanism provided by Couchbase and don't have to handle it ourselves. Finally, by using the `php-ext-couchbase` SDK, we don't need to use Moxi anymore because this is all handled for us underneath by the `libcouchbase` C library. There is only one limitation left: we still can't use map/reduce functionality because the documents aren't stored in JSON format. To accomplish this, we're getting rid of the PHP session handler altogether and implement it for ourselves.

Chances are that you're still using PHP 5.3 or lower. If this is the case, you want to either use the [old way](http://at.php.net/manual/en/class.sessionhandler.php) to define your session handler or just read on (because the method described below isn't bound to a specific PHP version).

## Implementing it on our own
When we implement the session handling mechanism on our own, we can reuse all insights from above and also use [JSON](http://nitschinger.at/Handling-JSON-like-a-boss-in-PHP) instead of the PHP object serialization. Imagine we're writing yet another PHP framework (don't do that...) and define a session interface like this:

	<?php
	namespace framework\sessions;

	interface SessionHandler {
		public function read($sessionId);
		public function write($sessionId, $data);
		public function delete($sessionId);
		public function check($sessionId);
	}
	?>

The `read`, `write` and `delete` methods should be clear what they're doing. The `check` method checks if the given key is found in the session store.

Here's again the full implementation, we'll talk it through afterwards:

	<?php
	namespace framework\sessions;

	use Couchbase;

	class CouchbaseSessionHandler implements SessionHandler {

		/**
		* Holds the Couchbase connection.
		*/
		protected $_connection = null;
		/**
		* The prefix to be used in Couchbase keynames.
		*/
		protected $_keyPrefix = 'session:';

		/**
		* Define a expiration time of 10 minutes.
		*/
		protected $_expire = 600;

		/**
		* Currently in development mode? Used for view loading.
		*/
		protected $_development = null;

		/**
		* Connect to Couchbase.
		*/
		public function __construct($host = '127.0.0.1:8091', $bucket = 'default', $dev = true) {
				$this->_development = $dev;
				$this->_connection = new Couchbase($host, $bucket);
				return $this->_connection ? true : false;
		}

		/**
		* Read the document for the given `sessionId`.
		*/
		public function read($sessionId) {
			$key = $this->_keyPrefix . $sessionId;

			$data = $this->_connection->get($key);
			return $data ? json_decode($data, true) : null;
		}

		/**
		* Write the data.
		*/
		public function write($sessionId, $data) {
			$key = $this->_keyPrefix . $sessionId;
			$data = $data + array('type' => 'session');

			return $this->_connection->set($key, json_encode($data), $this->_expire);
		}

		/**
		* Delete the data for the given `sessionId`.
		*/
		public function delete($sessionId) {
			$key = $this->_keyPrefix . $sessionId;

			return $this->_connection->delete($key);
		}

		/**
		* Check if a document exists.
		**/
		public function check($sessionId) {
			$key = $this->_keyPrefix . $sessionId;
			return $this->_connection->get($key) ? true : false;
		}

	}
	?>

This implementation is pretty basic and should give you just an idea what's possible. Since we're storing JSON in Couchbase, you can get fancy and implement the `read` and `check` method not only on the `sessionId` level, but also do fine-grained inspection of the keys inside the JSON document.

The code provided here is very similar to the example above, with only minor changes. The first one is that we open the connection on `__construct` and don't have a `gc` method since we don't need it here. Inside `read` and `write` we make use of `json_encode` and `json_decode` to convert the PHP payload into JSON. The `check` method works in a similar fashion to `read` but returns `true/false` instead of the actual data.

Let's run some code against our implementation:

	<?php
	use framework\sessions\CouchbaseSessionHandler;

	/**
	* Init the Handler.
	*/
	$handler = new CouchbaseSessionHandler();

	/**
	* Generate a Session ID for testing purposes.
	*/
	$sessionId = uniqid();

	/**
	* Store and retreive.
	*/
	$data = array('couchbase' => 'rocks');

	// Write some data
	$handler->write($sessionId, $data);

	// Returns the array
	var_dump($handler->read($sessionId));

	// Returns true
	var_dump($handler->check($sessionId));

	// Delete the document
	$handler->delete($sessionId);

	// Returns NULL
	var_dump($handler->read($sessionId));

	// Returns false
	var_dump($handler->check($sessionId));

	?>

If you comment out the `delete` part you can see that the document is stored as JSON in Couchbase. On the next request, you don't need the `write` call anymore since the data is loaded directly from Couchbase.

As a side note, my original idea was to implement a `clear` method that deletes all stored session documents. This is a perfect use-case for a view, but since the PHP extension with view support is still in a developer preview, [something went wrong](http://www.couchbase.com/issues/browse/PCBC-76). When this issue is resolved I'll write another blog post focusing on handling views. If you want to implement the `clear` method now, you can workaround `memcached`-style and maintain a separate document that just holds all keys of the current sessions. You can load this one and then delete all fetched keys.

## Final thoughts
There are lots of possibilities on how you can use Couchbase to store your PHP sessions. If you just want to switch to Couchbase on the backend and don't want to touch your code at all, the first option is for you. If you've used the built-in PHP session handling mechanisms and want to switch to a much more powerful storage engine, the second method may be a good fit. Finally, if you're using a PHP framework that uses its own adapters the last method provides the most flexibility and can be modified to suit all your needs.

Feel free to share your opinions below!