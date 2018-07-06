+++
date = "2011-07-09"
draft = false
title = "Testing the Lithium core for fun and profit"
description = "This post introduces you to Lithium core testing and shows you how to help out and make Lithium even more awesome."
tags = ["php","lithium","core","testing"]
slug = "Testing-the-Lithium-core-for-fun-and-profit"
+++

In modern web frameworks, testing is one of the most important pillars that ensure a clean, stable and extendable codebase. Testing support in Lithium was built in from the beginning and therefore it already features a great code coverage. The Lithium project aims to work out of the box on many platforms, including Microsoft Windows and its IIS, which was often neglected in the past. 

The main purpose of this post is to show you how core tests work and were the core team needs help to improve things further. At the end of this post you should be able to write new and extend existing test cases. A side effect of this is that you'll learn parts of the core from the inside out while you're testing them.

## Introducing core tests
Before you start writing your own tests, make sure the existing cases work as expected (without any `Exceptions` or `Errors`). For developing directly in the core, I'd recommend you a slightly different setup than the default one. You should clone the `framework` and the `lithium` repository in different directories so you can quickly exchange functionality and symlink stuff if you have to. My setup looks similar to this:

	- web
		- framework
		- libraries
			- lithium
			- li3_docs
			- manual
			- lithium_qa

Now tell your `framework` to look at the right path, hop into `app/config/bootstrap/libraries.php` and modify the `LITHIUM_LIBRARY_PATH`.

	define('LITHIUM_LIBRARY_PATH', dirname(dirname(LITHIUM_APP_PATH)) . '/libraries');

If you haven't already, make sure that the `app/resources` folder is writable.

Assuming you have your `DOCROOT` pointing to `web`, try to run the tests from the test dashboard located at `http://localhost/framework/test`. Ideally, your test output should look something like this:

<img src="http://nitschinger.at/img/core_testing_overview.png" title="Test Dashboard" />

If you have any failing tests, try to track down if it is your setup or some test cases are not working correctly. If you're unsure, you can always get feedback on #li3 at freenode.

## Understanding test filters
Now that existing tests are running as expected, take a look at the buttons on the top. The `Affected`, `Complexity`, `Coverage` and `Profiler` labeled buttons are called "test filters" and perform specific tasks on the selected group of test cases. 

<img src="http://nitschinger.at/img/core_testing_filters.png" title="Test Filters" />

As they are very important, let's go through them in more detail. Note that you need to turn on [Xdebug](http://xdebug.org/) to use them, but you should install this extension anyway in your development environment.

 - __Affected:__ The `affected` filter shows you what other tests depend on the currently tested class. This helps you to understand what impact a change to the current code would have and where to look out for possible errors.
 - __Complexity:__ The `complexity` filter calculates the [Cyclomatic Complexity](http://en.wikipedia.org/wiki/Cyclomatic_complexity) for each executed method and shows you the worst offenders. According to best practices, a cyclomatic complexity greater than 10 is "very complex" and should be refactored if possible. This of course depends heavily on the type of code but it provides a good overview on how complex the class is in general and where to look first to decrease complexity.
 - __Coverage:__ The `coverage` filters tries to find out which lines of the tested code got executed in your test. Lithium has the baseline of 85% test coverage for core classes which should ensure that most of the code is at least once tested somewhere. The current implementation of the coverage filter also shows some false-negatives so make sure to take the result with a grain of salt.
 - __Profiler:__ The `profiler` filter tells you how long it took to execute the tests and how much memory was used. You can use this to measure the performance of the code and check the impact of code modifications.

Make sure to get familiar with these filters as they are crucial in verifying and investigating your written tests.

## Investigating core tests
The best way to get familiar with core tests is to look at existing ones. Let's investigate one test case of the `ValidatorTest` class.

	/**
	 * Tests that new methods can be called on Validator by adding rules using Validator::add().
	 *
	 * @return void
	 */
	public function testAddCustomRegexMethods() {
		$this->assertNull(Validator::rules('foo'));

		Validator::add('foo', '/^foo$/');
		$this->assertTrue(Validator::isFoo('foo'));
		$this->assertFalse(Validator::isFoo('bar'));
		$this->assertTrue(in_array('foo', Validator::rules()));
		$this->assertEqual('/^foo$/', Validator::rules('foo'));

		$this->expectException("Rule `bar` is not a validation rule.");
		$this->assertNull(Validator::isBar('foo'));
	}

If you look closely, you can identify a pattern here that will come frequently. Before you test a specific functionality/value, make sure it isn't already set. You can do this with `assertFalse`, `assertNull` or maybe `assertTrue(empty(..))`. Next, perform the piece of code you want to test (like adding a custom regex rule here) and then perform checks on the entity again. A good rule of thumb is to test normal cases first and then exceptional cases (which often raise `Exceptions`). You can also break down complicated tests in smaller ones and add helper methods that get called from the main test case.

If your tests contain dependencies, make sure to initialize them in `setUp()` and remove/reset them in `tearDown()`. These methods get called before/after all tests in the test class and should leave the environment in a clean state. This prevents unexpected failures in later tests and also keeps the memory usage on a normal level. Here's an example of the `MongoDb` test class:

	public function setUp() {
		Connections::config(array('lithium_mongo_test' => $this->_testConfig));
		$this->db = Connections::get('lithium_mongo_test');
		$model = $this->_model;
		$model::config(array('key' => '_id'));
		$model::resetConnection(false);

		$this->query = new Query(compact('model') + array(
			'entity' => new Document(compact('model'))
		));
	}

	public function tearDown() {
		try {
			$this->db->delete($this->query);
		} catch (Exception $e) {}
		unset($this->query);
	}

The `setUp()` method initializes a test connection to the database and `tearDown()` makes sure that the test database gets deleted and the `$this->query` object is freed.

There are a lot more patterns that you'll learn while writing core tests, but this should give you a start.

## QA your code
To ensure a high-quality codebase, Lithium enforces a set of coding standards which can be found [here](https://github.com/UnionOfRAD/lithium/wiki/Coding-Standards). You don't have to know every part of the standard by heart, because Lithium provides a plugin called [lithium_qa](https://github.com/UnionOfRAD/lithium_qa). This plugin checks your code if it complies to the standards and shows you what parts of it are wrong. I recommend you to clone it into the `libraries` directory shown above. It is also a good idea to add the `li3` command to your `PATH` so your calls don't get too verbose.

	$ cd /usr/local/bin
	$ sudo ln -s /path/to/lithium/console/li3 li3

Let's assume we want to check classes in the `lithium\core` namespace:

	$ cd lithium_qa
	$ li3 syntax ../lithium/core
	[Passed ] syntax check of `core/Object.php`
	[Passed ] syntax check of `core/Adaptable.php`
	[Failed ] syntax check of `core/StaticObject.php`
	59| 101| Maximum line length exceeded
	[Passed ] syntax check of `core/Environment.php`
	[Passed ] syntax check of `core/ConfigException.php`
	[Passed ] syntax check of `core/ClassNotFoundException.php`
	[Passed ] syntax check of `core/NetworkException.php`
	[Passed ] syntax check of `core/ErrorHandler.php`
	[Passed ] syntax check of `core/Libraries.php`

There you can see that each core class passed the test except the `StaticObject` class which has one line (59) that is too long. Now you can edit that file and modify it accordingly. If you're not sure what the output means, head back to the [official standard](https://github.com/UnionOfRAD/lithium/wiki/Coding-Standards) and look for the corresponding entry.

Please make sure that all code you want to see in the core respects these standards. You can also use it to test your own code as well!

## Getting help
If you're new to code testing in Lithium, make sure to read the official guide in the [manual](http://lithify.me/docs/manual/10_testing/readme.wiki) first. Next, check out the main [testing ticket on GitHub](https://github.com/UnionOfRAD/lithium/issues/17) which is constantly updated and shows what classes need testing. If you need help on testing the core, make sure to join #li3-core on freenode and ping one of the core developers. You can also contact me directly and I'll make sure to help you out or point you in the right direction.

## Wrapping up
You should now be able to run the core tests, extend and write your own and also pass your code through the official code quality assurance tool. Please help out and fork the core, run it on your machine, check for errors, extend current tests, write new ones and correct QA issues. If we all work together, I'm sure we can make Lithium the best tested web framework out there. If you want to see more information regarding this topic on my blog, comment down below and I'll see what I can provide.

Keep on coding!