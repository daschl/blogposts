+++
date = "2013-02-19"
draft = false
title = "A Journey on Avoiding Nulls in PHP"
description = "What about avoiding nulls completely in your PHP application? Inspired by Scala and Google Guava, this post shows you how to do it in PHP."
tags = ["php"]
slug = "A-Journey-on-Avoiding-Nulls-in-PHP"
+++

Let's face it: nulls are a hassle and lead to exceptions and inconsistent application state. [Tony Hoare](http://en.wikipedia.org/wiki/Tony_Hoare), the inventor of QuickSort, even calls it his [billion dollar mistake](http://qconlondon.com/london-2009/presentation/Null+References:+The+Billion+Dollar+Mistake).

While every developer has kind of accepted their existence, they are suddenly there when we'd desperately need them to not show up. How often did you write`if($obj === null)` in your PHP code? Can't there be a better, more elegant and fault-tolerant solution to the problem?

It turns out that for example in [Scala](http://www.scala-lang.org/), you can avoid nulls by using the [Option](http://www.scala-lang.org/api/current/index.html#scala.Option) class. It allows you to express in a clear way that the result of a - lets say - method can either be good (a reference to a valid object) or bad. In PHP, bad often means null. Scala then consequently provides method to access this object safely. You as the developer can choose at which point you want to get the value out, either by accessing it directly (and catching exceptions) or provide a sensible default value.

In Scala, you write code like this:

	/* An option with a value */
	val a: Option[Int] = Some(5)
	/* An option with no value */
	val b: Option[Int] = None

	/* Returns 5 */
	a.getOrElse(0)
	/* Returns 10 */
	b.getOrElse(10)

You can also use helpful methods like `isDefined()` to peek into the object and see whats going on at runtime. All this is much more expressive than juggling around with a possible `null` value.

Java has the same issues as PHP, and it hurt Google so much that they integrated the concept of an [Optional](http://code.google.com/p/guava-libraries/wiki/UsingAndAvoidingNullExplained) directly into the core of their [Google Guava](http://code.google.com/p/guava-libraries/) Java library collection. I really like their code and use it in my Java projects, so I decided it would be an interesting project to port this to PHP. Since PHP isn't a statically typed language like Java, concepts like generics have been removed and some methods renamed (they look more like Scala's now), but in essence its the same.

## Usage
Before digging into how it works, let's look at how we can use it. We only need to work with three classes called `Optional`, `Present` and `Absent`. `Optional` is the base class, `Present` and `Absent` inherit from it and provide the method bodies for their appropriate behavior.

The main idea is to fail fast when null is discovered and afterwards have a safe way of working with our value. Creating a `Optional` works through the static `of` method. We then have a range of methods available to inspect and fetch the output:

	<?php
	$possible = Optional::of(5);
	var_dump($possible->isPresent()); // bool(true)
	var_dump($possible->get()); // int(5)
	var_dump($possible->getOrElse(99)); // int(5)
	var_dump($possible->getOrNull()); // int(5)
	?>

If we're trying to call `of` with `null`, a `NullPointerException` is raised immediately. We can use the `fromNullable` method to handle this case safely:

	<?php
	$possible = Optional::of(null); // Throws 'NullPointerException' with message 'Unallowed null in reference found.'
	$possible = Optional::fromNullable(null);
	var_dump($possible->isPresent()); // bool(false)
	var_dump($possible->get()); // Throws IllegalStateException
	var_dump($possible->getOrElse(99)); // int(99)
	var_dump($possible->getOrNull()); // NULL
	?>

We can even check the equality of the contained Optionals:

	<?php
	$val1 = Optional::fromNullable(5);
	$val2 = Optional::fromNullable(4);
	$val3 = Optional::fromNullable(4);

	var_dump($val1->equals($val2)); // bool(false)
	var_dump($val2->equals($val3)); // bool(true)
	?>

Now this simple examples are easy to grasp and this is a good thing. It's not about complexity here, it's about making it obvious that a value may or may not be null and handling it appropriately.

If you're accessing a method of a library, you need to check the return value every time if it's null to make sure you're not in an invalid state. If it's clear that the library is returning an instance of `Optional`, you know what to expect and how to deal with it appropriately.

## Implementation
Let's look at how this implemented.

	<?php
	abstract class Optional {
		
		private function __construct() {}

		/**
		 * Returns an instance with no contained reference.
		 */
		public static function absent() {
			return Absent::instance();
		}

		/**
	     * Returns an Optional instance containing the given non-null reference.
	     */
		public static function of($reference) {
			return new Present(static::checkNotNull($reference));
		}

		/**
		 * If reference is non-null, returns an Optional instance containing
		 * that reference; otherwise returns Absent.
		 */
		public static function fromNullable($reference) {
			return $reference === null ? static::absent() : new Present($reference);
		}

		public abstract function isPresent();
		public abstract function get();
		public abstract function getOrElse($defaultValue);
		public abstract function getOrNull();
		public abstract function equals($object);

		/**
		 * Make sure the passed reference is not null.
		 */
		protected static function checkNotNull($reference, $message = null) {
			if($message === null) {
				$message = "Unallowed null in reference found.";
			}

			if($reference === null) {
				throw new NullPointerException($message);
			}
			return $reference;
		}


	}
	?>

This abstract class defines all of the methods we'll implement in `Present` and `Absent`, as well as all the static methods that you can call directly (especially the `of` and `fromNullable` methods). Note that in the appropriate places, `checkNotNull` is called which will throw a `NullPointerException` if a null is found. This adheres to the fail fast principle.

Now let's implement the `Absent` class, which represents the state where no reference is present (for example a null was passed in through `fromNullable`).

	<?php
	class Absent extends Optional {

		private static $instance;

		private function __construct() {}

		public function isPresent() {
			return false;
		}

		public function get() {
			throw new IllegalStateException("Optional->get() cannot be called on an absent value");
		}

		public function getOrElse($defaultValue) {
			$message = "use Optional->orNull() instead of Optional->or(null)";
			return static::checkNotNull($defaultValue, $message);
		}

		public function getOrNull() {
			return null;
		}

		public function equals($object) {
			return $object === $this;
		}

		protected static function instance() {
			if(static::$instance == null) {
				return static::$instance = new Absent();
			}
			return static::$instance;
		}

	}
	?>

The `$instance` variable is managed here through the singleton pattern. We only need one instance of `Absent`, this saves memory and CPU time. The other methods just implement the expected behaviour of something that is not available.

Finally, we can implement the `Present` class, which actually stores our instance if available.

	<?php
	class Present extends Optional {

		private $reference;

		protected function __construct($reference) {
			$this->reference = $reference;
		}

		public function isPresent() {
			return true;
		}

		public function get() {
			return $this->reference;
		}

		public function getOrElse($defaultValue) {
			$message = "use Optional->orNull() instead of Optional->or(null)";
			static::checkNotNull($defaultValue, $message);
			return $this->reference;
		}

		public function getOrNull() {
			return $this->reference;
		}


		public function equals($object) {
			if($object instanceof Present) {
				return $this->reference === $object->get();
			}
			return false;
		}
	}
	?>

It looks the same as the `Absent` class, but manages to return our instance if necessary and implements the `equals` method to actually check with the given object.

Edit: Rasmus Schultz pointed out that he uses the ternary operator for lightweight null checks, you may want to [check out](http://blog.mindplay.dk/2013/02/the-or-else-operator-in-php-53.html) his post too!

## Summary
As you can see, the adoption of the `Optional` pattern can greatly increase the expressiveness of your code, by consequently avoiding null and its associated pitfalls. It also makes your code less error-prone, because you can't stumble upon nulls when using this pattern consequently. 

This is kind of an experimental implementation, but I think putting this together in a utility library could drive the adoption in the PHP community. I'm curious what you think about the concept and if it makes sense for you and your applications!