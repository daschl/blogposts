+++
date = "2012-05-22"
draft = false
title = "A primer on PHP exceptions"
description = "This post provides a whirlwind tour through the architecture and implementation of PHP exceptions."
tags = ["php","exceptions"]
slug = "A-primer-on-PHP-exceptions"
+++

# Preface
Exceptions are and should be an integral part of any general purpose programming language. PHP introduced them long ago (with the release of PHP 5 or 5.1), but it still seems that many of the concepts are not fully understood or ignored by the community. This post aims to be a solid introduction to exception architecture, handling and testing. At the end of the post you should be able to know when to raise an exception and how it should look like.

Of course, there are many opinions on a topic like this. If you have constructive feedback, feel free to comment below. Let's get started!

# Architecture & Best-Practices
Here's a golden rule: use exceptions for what their name says: exceptional situations. Don't use exceptions to control the flow of your application logic (like substituting if-statements or to control loops). This not only leads to confusion, but also my slow down your code.

Edit: Due to some confusions on [reddit](http://www.reddit.com/r/PHP/comments/tz5jw/a_primer_on_php_exceptions/), I think I need to clarify what exceptional situations are. When they occur, your code/request normally can't continue correctly. So, if your database is not ready and you rely on it, you can't continue. If you validate a form and a validation for the "username" field fails, you can continue and present the user with an error message tied to that form field. This depends heavily on your use case, but I hope you get the point.

In PHP, you have no way of finding out what exceptions the called method may throw if it isn't documented somewhere or you can look at the code (in contrast, Java provides an explicit "throws" notation in the method definiton). So if you want to make sure that your code catches everything, you need to catch exceptions from the "Exception" class (in PHP, all exceptions need to have the "Exception" class or a descendant of it).

Here's another tip that might come in handy if you develop code that includes third-party libraries: make sure to "normalize" your exceptions as soon as possible. This allows you to handle them better up the stack. What do I mean by that? Consider the following example:

You are writing a database abstraction layer (not that there aren't already enough...) and you need to integrate third party extensions that handle the lowlevel/protocol stuff. We want to implement two databases: "Foobase" and "BarSQL". When each driver fails to open a connection (for example, we've provided an invalid hostname), it raises an exception. Foobase raises an  "FoobaseConnectionException" and BarSQL raises an "InvalidHostnameException". This is a nightmare to handle up the stack if we want to log them accordingly or write those exceptions to a third-party logging service. Here's what you can do: catch them early and raise a "normalized" exception like "NetworkException" with the actual exception as the message. We'll see later how to do this.

While providing different exception classes for different situations is fine, make sure not to over engineer it. In my opinion, you should plan your exception classes like you plan the overall architecture of your library. This way, you're also forced to think about all possible exceptional situations that may occur and give you a hint or two where you should be careful. Also, it is often a good idea to refactor dangerous code in their own helper methods so they can be better tested and don't mess up the rest of your code.

After a bit of theory, let's get to something more practical. How does the exception class hierarchy in PHP look like?

	- Exception
		- ErrorException
		- LogicException
			- BadFunctionCallException
				- BadMethodCallException
			- DomainException
			- InvalidArgumentException
			- LengthException
			- OutOfRangeException
		- RuntimeException
			- OutOfBoundsException
			- OverflowException
			- RangeException
			- UnderflowException
			- UnexpectedValueException

At the root of the class hierachy, we have the [Exception](http://us.php.net/manual/en/class.exception.php) class. All exceptions have to inherit from it (or a subclass), otherwise you can't throw it. Down the hierachy, we have the [ErrorException](http://us.php.net/manual/en/class.errorexception.php), the [LogicException](http://us.php.net/manual/en/class.logicexception.php) and the [RuntimeException](http://us.php.net/manual/en/class.runtimeexception.php). The `ErrorException` is used to transform errors into exceptions (we'll see later how). The other ones are the top two exception objects in the [SPL](http://us.php.net/manual/en/book.spl.php) hierachy. All other built-in exceptions inhiert from one of them. I won't describe them here, since their names are pretty expressive. Just make sure to throw one of them if they fit your needs. Don't try to invent similar ones when they are already defined. This helps you to keep your exception architecture clean and focused.

All built-in exception classes don't use a specific namespace (most of them were introduced before PHP 5.3), but you can namespace them similar to normal classes. You're forced to do this if you use a standard like [PSR-0](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-0.md). For example, [Lithium](http://lithify.me) has the following exception defined:

	<?php
	namespace lithium\net\http;

	class RoutingException extends \RuntimeException {
		protected $code = 500;
	}
	?>

You can infer from the context that it has something to do with a `Router` and is thrown during runtime. The `Router` class who throws it is placed in the same namespace. Notice how the `RuntimeException` is prefixed with a backslash - this indicates that we want to use a class from the "root" namespace. The method who uses is looks like this:

	<?php
	namespace lithium\net\http;
	use lithium\net\http\RoutingException;

	class Router extends \lithium\core\StaticObject {	
		public static function match($url = array(), $context = null, array $options = array()) {
			// ...
			// code that handles url matching...
			// ...
			throw new RoutingException("No parameter match found for URL `{$url}`.");
		}
	}
	?>

PHP doesn't provide a way to define what kind exceptions may be thrown, so make sure to document them correctly. Most documentation tools like [phpDocumentor](http://www.phpdoc.org/) provide a `@throws` annotation. For example: 

	<?php
	class MyClass {
	
		/**
		 * This method just throws an exception.
		 *
		 * @throws RuntimeException
		 */
		public function foo() {
			throw new RuntimeException("Ooops...");
		}
		
	}
	?>

# PHP Syntax
We've already seen the basic syntax of throwing exceptions, but here is the formal method definition:

	throw new Exception($message, $code, $previous);

All arguments are optional, but you should at least provide a descriptive message. The `code` argument is an arbitary error code that you can provide. For example, you can define your own codes or use them with HTTP status codes. If you are rethrowing a exception, you can also provide the instance through the `previous` argument. This way, both exceptions get raised to the calling method.

After throwing it, you need to catch it somewhere. PHP provides the familiar try/catch construct, but without a `finally` block like in Java.

	try {
		$object = new IThrowExceptions();
	} catch(Exception $e) {
		echo "Caught " . $e->getMessage();
	}

The code inside the `try` block is evaluated, and if an exception is thrown the `catch` block tries to handle it. If the exception can't be handled, it bubbles up the stack. If the exception doesn't get handled in your code, PHP terminates. If you expect different exceptions to be thrown, you can catch more than one at the same time:

	try {
		$foo = new Foo();
		$foo->bar();
	} catch(InvalidArgumentException $e) {
		echo "Just InvalidArgumentExceptions";
	} catch(RuntimeException $e) {
		echo "Just RuntimeExceptions";
	} catch(Exception $e) {
		echo "I catch everything";
	}

The last `catch(Exception $e)` is used when no previous catch succeeded. Make sure to go from special to general, because when you move the `Exception` block to the top, the exception would never sink down to the `RuntimeException`.

Alright, let's get back to our example from the beginning. Imagine we get a lot of custom exceptions that all have the same meaning. We want to normalize them, so that only one specific type bubbles up the stack. In our example, no matter what kind of exception we receive during `_connect`, we want to throw a `NetworkException`. It's pretty easy to rethrow the exception:

	<?php
	/**
	 * Our custom Exceptions.
	 */
	class FoobaseConnectException extends Exception {}
	class BarSQLException extends Exception {}
	class NetworkException extends Exception {}
	
	/**
	 * A sample parent datasource.
	 */ 
	abstract class Datasource {
		public function connect() {
			try {
				$this->_connect();
			} catch(Exception $e) {
				$message = "Could not connect: " . $e->getMessage();
				throw new NetworkException($message, $e->getCode());
			}
		}
		abstract protected function _connect();
	}

	/**
	 * Foobase datasource.
	 */
	class Foobase extends Datasource {
		protected function _connect() {
				throw new FoobaseConnectException("Holy moly, no DB available");
		}
	}

	/**
	 * BarSQL datasource.
	 */
	class BarSQL extends Datasource {
		protected function _connect() {
				throw new BarSQLException("Y U NO SQL SERVICE?");
		}
	}
	?>

If you execute the following code, a `'NetworkException' with message 'Could not connect: Holy moly, no DB available'` is raised:

	<?php
	$foobase = new Foobase();
	$foobase->connect();
	?>

We not only rethrow a different exception, but also incorporate the original message to provide a more detailed error report.

The exception classes provide more accessor-methods like reading the current line or file, check out the docs [here](http://php.net/manual/de/class.exception.php). In our example above we also added our own exception classes. If you like, you can also add methods and call them later (maybe you need custom stack traces or want to add more information to the exception).

The last thing that comes in very handy are error handlers. Recall the `ErrorException` from the beginning? Here's what you can do with it (taken from [here](http://us.php.net/manual/en/class.errorexception.php)):

	function exception_error_handler($errno, $errstr, $errfile, $errline ) {
		throw new ErrorException($errstr, 0, $errno, $errfile, $errline);
	}
	set_error_handler("exception_error_handler");
	
	/* Trigger exception */
	strpos();

PHP throws lots of errors and warnings from built-in methods like `strpos`. The problem is that you can't do anything with them in your code until you translate them into exceptions. That's 
why the `ErrorException` was implemented. Notice how different the constructor is to the `Exception` class, because it mirrors all variables that are passed to the error handler. This allows 
you to capture and handle everything that happens during code execution in a uniform way, being it exceptions or errors.

# Testing Exceptions
Testing exceptions may seem daunting at first, since you need to verfiy that your code fails. This usually means that you need to write unit tests with try/catch blocks and then do 
some kind of assertion inside the catch-block like this (just assume we have some arbitary `assert` functions defined):

	<?php

	class Foo {
		public function bar() {
			throw new RuntimeException("My Message", 123);
		}
	}

	$foo = new Foo();
	try {
		$foo->bar();
	} catch(RuntimeException $e) {
		assertEqual($e->getMessage(), "My Message");
		assertEqual($e->getCode(), 123);
	}

	?>

PHP 5.3 provides us with a very nice addition to the language: [closures](http://php.net/manual/en/functions.anonymous.php). We can write a handy method that lets us test this kind of code very elegantly. Here's a simple helper method:

	function assertException($expected, $closure) {
		try {
			$closure();
			echo "Exception expected, but not thrown";
		} catch (Exception $e) {
			if($e->getMessage() == $expected) {
				echo "Correctly thrown";
			} else {
				echo "Exception thrown, but with a different message.";
			}
		}
	}

We run the closure inside the try/catch block and check if the correct message is raised. Now we can test a code snippet like this:

	<?php
		class Foo {
			public function bar() {
				throw new RuntimeException("My Message", 123);
			}
		}
		
		// prints "Correctly Thrown"
		assertException("My Message", function() {
			$foo = new Foo();
			$foo->bar();
		});
	?>

Of course, in a real test suite you want to replace the `echo` statements with debug output and add more intelligence. Look [here](https://github.com/UnionOfRAD/lithium/blob/master/test/Unit.php#L521) for a practical implementation. This allows you to test exceptions in a natural and convenient way. Here's a real unit test from a [Couchbase datasource for Lithium](https://github.com/daschl/li3_couchbase):

	public function testConnect() {
		$result = new Couchbase($this->_dbConfig);
		$this->assertTrue($result->isConnected());
		$this->assertTrue(is_string($result->connection->getVersion()));

		$this->assertException('/Unknown host/', function() {
			$result = new Couchbase(array('host' => 'invalidHostname'));
		});
	}

You can see that the exception assertion is integrated tightly after the normal tests and no try/catch blocks are used. Pretty neat, right?

Edit: David Thalmann thankfully pointed out that you can also test exceptions through annotations on your test methods. [Here's](http://www.phpunit.de/manual/3.7/en/writing-tests-for-phpunit.html#writing-tests-for-phpunit.exceptions) how to do that in [PHPUnit](http://www.phpunit.de). 

# Wrapping Up
While exceptions are part of nearly every language, using them correctly and not throwing something just for the sake of it is not easy and requires some thought. Make sure to think of your exceptions while you design the API, test and document them correctly. A lot of frameworks do a very good job on this, so keep your eyes open and prepare yourself to learn something new.

Remember: You gotta catch em' all!