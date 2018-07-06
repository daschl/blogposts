+++
date = "2013-01-08"
draft = false
title = "Caching Doctrine Entities with Couchbase"
description = "This blog post gives you an introduction on how you can speed up your Doctrine ORM layer by using Couchbase Server 2.0 as a simple, fast and elastic cache in front of your entities."
tags = ["doctrine","couchbase","php","cache"]
slug = "Caching-Doctrine-Entities-with-Couchbase"
+++

## Motivation
As part of our ongoing efforts to make Couchbase more integrated with frameworks and libraries, we added caching support for the [Doctrine ORM](http://www.doctrine-project.org/). Recently, the pull request has been merged into the master branch and is scheduled to be published along with the [2.4](http://doctrine-project.org/jira/browse/DCOM/fixforversion/10327) release.

Caching can either be used standalone (through the API provided by [doctrine/common](http://www.doctrine-project.org/projects/common.html)) or integrated with the ORM functionality. We'll look at both variants through simple examples, a good documentation can also be found [here](http://docs.doctrine-project.org/en/latest/reference/caching.html). Note that at the time of writing, the CouchbaseCache is not mentioned as a caching driver because the documentation still needs to be updated.

Since 2.4 has not been released yet, we need to work against the `2.4.x-dev` branch. We'll be using [Composer](http://getcomposer.org/) to fetch our dependencies, so just need change the version number if you want to pin it down to 2.4 later.

## Simple Caching
Our first example shows how the caching API can be used directly. If you are familiar with the Couchbase API, you may think that it's more or less just a different API with the same (and maybe less) semantics, but the point is that it uses the Doctrine Cache API interface and as a result you can switch between different caching implementations very easily.

Create a directory called `couchbase-doctrine-simple` with the following `composer.json` inside:

	{
		"require": {
			"doctrine/common": "2.4.x-dev",
			"ext-couchbase": "1.1.x"
		}
	}

This installs the [doctrine/common](https://packagist.org/packages/doctrine/common) package and also makes sure that we have the `couchbase.so` extension in place. If you haven't installed the Couchbase PHP extension already, head over to the [official website](http://www.couchbase.com/develop/php/current) and install it based on the tutorial and the docs.

Create a `index.php` with the following content (we'll break up the code afterwards):

	<?php
	// 0: Composer Autoloader
	require 'vendor/autoload.php';

	// 1: Open the Couchbase Connection
	$couchbase = new Couchbase("127.0.0.1", "", "", "default");

	// 2: Instantiate the Driver & Inject the Connection
	$cacheDriver = new \Doctrine\Common\Cache\CouchbaseCache();
	$cacheDriver->setCouchbase($couchbase);

	// 3: Execute your commands!
	$key = "my-cache-item";

	if(!$cacheDriver->contains($key)) {
		$cacheDriver->save($key, "my_data");
	} else {
		echo $cacheDriver->fetch($key);
	}
	?>

First, we need to bootstrap the composer autoloader so we don't have to write all `require` statements on our own. The next thing we need to do is actually connect to the Couchbase cluster:

	// 1: Open the Couchbase Connection
	$couchbase = new Couchbase("127.0.0.1", "", "", "default");

Here, we're connecting to a node in the cluster which points at `localhost`, but you can pass in an array of nodes as well. We connect to the `default` bucket, which has no password. Now that we have our connection established, we can instantiate the cache driver and inject our Couchbase client:

	// 2: Instantiate the Driver & Inject the Connection
	$cacheDriver = new \Doctrine\Common\Cache\CouchbaseCache();
	$cacheDriver->setCouchbase($couchbase);

From here on, the API is the same for all cache drivers. The following code checks if the cache contains a key. If it is present, it prints out the document but if it isn't it creates a new one. This is a very simple example but shows how you can start to use Couchbase caching in your own projects with just a few lines of bootstrapping!

Aside from these three methods, there is also a `delete` method available. Finally, you can pass an optional third param on `save` with a `$lifeTime` so that the cache item vanishes automatically.

Since Couchbase Server doesn't care what you store, you can also save and fetch any kind of datatype (aside from resources):

	$cacheDriver->save($key, array('foo' => 'bar'));
	var_dump($cacheDriver->fetch($key));

Note that when you use the driver at this level, try to store JSON strings when you can (use `json_encode`/`json_decode` on your datastructures) This way you can take advantage of the brand new view engine inside Couchbase Server 2.0. You can always just store serialized objects as well (like we need to do with ORM integration) since for Couchbase Server it's just a byte stream. 

We can now build on this foundation and see how this works with ORM integration.

## ORM Integration
Create a new directory called `couchbase-doctrine-orm` with the following `composer.json`:

	{
		"require": {
			"doctrine/orm": "2.4.x-dev",
			"doctrine/dbal": "2.4.x-dev",
			"doctrine/common": "2.4.x-dev",
			"ext-couchbase": "1.1.x"
		},
		"autoload": {
			"psr-0": {
				"Entities": "src/"
			}
		}
	}

This time our `composer.json` file is a little bit longer, because we need to define all of our dependencies by hand (since we don't want to work against the stable release). Since we need to define Doctrine Entities we pass the composer autoloader the custom directory (`src/`).

The next thing we need is our actual entity that we want to manage through Doctrine. Go ahead and create a `Person.php` file inside the `src/Entities` directory with the following content:

	<?php

	namespace Entities;

	/** @Entity */
	class Person {

		/**
		 * @Id @Column(type="integer") @GeneratedValue(strategy="AUTO")
		 */
		private $id;

		/** @Column(type="string") */
		private $firstname;

		/** @Column(type="string") */
		private $lastname;

		public function setFirstname($firstname) {
			$this->firstname = $firstname;
		}

		public function getFirstname() {
			return $this->firstname;
		}

		public function setLastname($lastname) {
			$this->lastname = $lastname;
		}

		public function getLastname() {
			return $this->lastname;
		}

	}

	?>

This is a very simple Doctrine Entity that has some properties and also a autogenerated `ID` field. I'm going to use [SQLite](http://www.sqlite.org/) in the following example, but feel free to use MySQL or any other relational database that you have available.

To wire everything together, we're going to create a `index.php` file in the root directory of the project. Again, here is the full content and we're going to break it apart afterwards:

	<?php
	// Composer autoloader.
	$loader = require 'vendor/autoload.php';

	/**
	 * Initialize Couchbase & the Cache.
	 */
	$couchbase = new Couchbase("127.0.0.1", "", "", "default");
	$cacheDriver = new \Doctrine\Common\Cache\CouchbaseCache();
	$cacheDriver->setCouchbase($couchbase);


	/**
	 * Initialize the Entity Manager.
	 */
	$paths = array(__DIR__ . '/src/Entities/');
	$isDevMode = true;
	$dbParams = array(
		'driver' => 'pdo_sqlite',
		'user' => 'root',
		'password' => '',
		'path' => __DIR__ . '/cbexample.sqlite'
	);

	$config = \Doctrine\ORM\Tools\Setup::createAnnotationMetadataConfiguration($paths, $isDevMode, null, $cacheDriver);
	$em = \Doctrine\ORM\EntityManager::create($dbParams, $config);

	/**
	 * Work with our Entities.
	 */
	$person = new \Entities\Person();
	$person->setFirstname("Michael");
	$person->setLastname("Nitschinger");

	$em->persist($person);
	$em->flush();

	// Query with Result Cache
	$query = $em->createQuery('select p from Entities\Person p');
	$query->useResultCache(true);
	$results = $query->getResult();

	?>

Since this may be a lot to grasp, let's break it into smaller sized chunks.

	$couchbase = new Couchbase("127.0.0.1", "", "", "default");
	$cacheDriver = new \Doctrine\Common\Cache\CouchbaseCache();
	$cacheDriver->setCouchbase($couchbase);

After our bootstrapping the autoloader, we're initializing the cache driver. You already know what this means because we've used the same code in the simple example before.

	$paths = array(__DIR__ . '/src/Entities/');
	$isDevMode = true;
	$dbParams = array(
		'driver' => 'pdo_sqlite',
		'user' => 'root',
		'password' => '',
		'path' => __DIR__ . '/cbexample.sqlite'
	);

	$config = \Doctrine\ORM\Tools\Setup::createAnnotationMetadataConfiguration($paths, $isDevMode, null, $cacheDriver);
	$em = \Doctrine\ORM\EntityManager::create($dbParams, $config);

The `\Doctrine\ORM\EntityManager` is one of the major building blocks inside Doctrine and needs to be initialized accordingly. Therefore, we need to provide it a valid configuration. Here we're going to use annotations (as seen on the Doctrine Entity, but you can also do it through XML or YAML). We also need to provide our database connection and the path to the entities. The important part here is that we pass the `$cacheDriver` to the factory method. This automatically initializes our `CouchbaseCache` to be used for all kinds of caching (Query, Metadata and Result caching).

Now we can go ahead and create a record:

	$person = new \Entities\Person();
	$person->setFirstname("Michael");
	$person->setLastname("Nitschinger");

	$em->persist($person);
	$em->flush();

Afterwards, we can fetch it back through a query:

	$query = $em->createQuery('select p from Entities\Person p');
	$query->useResultCache(true);
	$results = $query->getResult();

Note that we explicitely tell it to cache this query result for us (by default, result caching won't be used). If open the browser and point it to your Couchbase Server 2.0 management UI, you can see that Doctrine did create lots of documents behind the scenes. These are subsequently used to boost your application performance.

## Summary
As you can see, using Couchbase as a Cache for Doctrine is not hard. You just need to initialize it and pass it into the configuration. From this point on, everything happens behind the scenes. And don't forget that you not only get exceptional performance, but also persistence, scalability and all the cool stuff that Couchbase Server provides out of the box.

If you have any questions or input, please let me know in the comments! Finally, thanks to [Marco Pivetta](https://twitter.com/Ocramius) for helping me debug an [issue](https://github.com/doctrine/common/pull/240) with ORM integration!