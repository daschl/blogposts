+++
date = "2012-06-06"
draft = false
title = "Handling JSON like a boss in PHP"
description = "After reading this post, you will be able to understand, work and test JSON code with PHP."
tags = ["php","json"]
slug = "Handling-JSON-like-a-boss-in-PHP"
disqus_identifier = "613cd50246038baeb4f8fd059ac7fa8246e5152e6347e820959ba5da1e1499fa9b46ec7a45312fd810fb2ba34da37a63d2f275ccb084b3dd13da63ad062b0197"
+++

There are already lots of tutorials out there on handling JSON with PHP, but most of them don't go much deeper than throwing an array against [json_encode](http://php.net/manual/en/function.json-encode.php) and hoping for the best. This article aims to be a solid introduction into JSON and how to handle it correctly in combination with PHP. Also, readers who don't use PHP as their programming language can benefit from the first part that acts as a general overview on JSON.

JSON (JavaScript Object Notation) is a data exchange format that is both lightweight and human-readable (like XML, but without the bunch of markup around your actual payload). Its syntax is a subset of the JavaScript language that was standardized in [1999](http://www.ecma-international.org/publications/files/ECMA-ST/Ecma-262.pdf). If you want to find out more, visit the [official website](http://www.json.org/).

The cool thing about JSON is that you can handle it natively in JavaScript, so it acts as the perfect glue between server- and client-side application logic. Also, since the syntactical overhead (compared to XML) is very low, you need to transfer less bytes of ther wire. In modern web stacks, JSON has pretty much replaced [XML](http://en.wikipedia.org/wiki/XML) as the de-factor payload format (aside from the Java world I suppose) and client side application frameworks like [backbone.js](http://backbonejs.org/) make heavy use of it inside the model layer.

Before we start handling JSON in PHP, we need to take a short look at the JSON specification to understand how it is implemented and what's possible.

## Introducing JSON
Since JSON is a subset of the JavaScript language, it shares all of its language constructs with its parent. In JSON, you can store unordered key/value combinations in objects or use arrays to preserve the order of values. This makes it easy to parse and to read, but also has some limitations. Since JSON only defines a small amount of data types, you can't transmit types like dates natively (you can, but you need to transform them into a string or a unix timestamp as an integer).

So, what datatypes does JSON support? It all boils down to strings, numbers, booleans and null. Of course, you can also supply objects or arrays as values.

Here's a example JSON document:

	{
		"title": "A cool blog post",
		"clicks": 4000,
		"children": null,
		"published": true,
		"comments": [
			{
				"author": "Mister X",
				"message": "A really cool posting"
			},
			{
				"author": "Misrer Y",
				"message": "It's me again!"
			}
		]
	}

It contains basically everything that you can express through JSON. As you can see no dates, regexes or something like that. Also, you need to make sure that your whole JSON document is encoded in [UTF-8](http://en.wikipedia.org/wiki/UTF-8). We'll see later how to ensure that in PHP. Due to this shortcomings (and for other good reasons) [BSON](http://bsonspec.org/)(Binary JSON) was developed. It was designed to be more space-efficient and provides traversability and extensions like the date type. Its most prominent use case is [MongoDB](http://www.mongodb.org/), but honestly I never came across it somewhere else. I recommend you to take a short look at the specification if you have some time left, since I find it very educating.

Since PHP has a richer type handling than JSON, you need to prepare yourself to write some code on both ends to transform the correct information apart from the obligatory encoding/decoding step. For example, if you want to transport date objects, you need to think if you can just send a unix timestamp over the wire or maybe use a preformatted date string (like [strftime](http://php.net/manual/en/function.strftime.php)).

## Encoding JSON in PHP
Some years ago, JSON support was provided through the [json pecl extension](http://pecl.php.net/package/json). Since PHP 5.2, it is included in the core directly, so if you use a recent PHP version you should have no trouble using it.

Note: If you run an older version of PHP than 5.3, I recommend you to upgrade anyway. PHP 5.3 is the oldest version that is currently supported and with the latest PHP [security bugs](http://www.php.net/archive/2012.php#id2012-02-02-1) found I would consider it critical to upgrade as soon as possible.

Back to JSON. With [json_encode](http://php.net/manual/en/function.json-encode.php), you can translate anything that is UTF-8 encoded (except resources) from PHP into a JSON string. As a rule of thumb, everything except pure arrays (in PHP this means arrays with an ordered, numerical index) is converted into an object with keys and values.

The method call is easy and looks like the following:

        json_encode(mixed $value, int $options = 0);

An integer for options you might ask? Yup, that's called a [bitmask](http://en.wikipedia.org/wiki/Mask_(computing)). We'll cover them in a separate part a little bit later. Since these bitmask options change the way the data is encoded, for the following examples assume that we use defaults and don't provide custom params.

Let's start with the basic types first. Since its so easy to grasp, here's the code with short comments on what was translated:

	<?php
	// Returns: ["Apple","Banana","Pear"]
	json_encode(array("Apple", "Banana", "Pear"));

	// Returns: {"4":"four","8":"eight"}
	json_encode(array(4 => "four", 8 => "eight"));

	// Returns: {"apples":true,"bananas":null}
	json_encode(array("apples" => true, "bananas" => null));
	?>

How your arrays are translated depends on your indexes used. You can also see that `json_encode` takes care of the correct type conversion, so `booleans` and `null` are not transformed into strings but use their correct type. Let's now look into objects:

	<?php
	class User {
		public $firstname = "";
		public $lastname  = "";
		public $birthdate = "";
	}

	$user = new User();
	$user->firstname = "foo";
	$user->lastname  = "bar";

	// Returns: {"firstname":"foo","lastname":"bar"}
	json_encode($user);

	$user->birthdate = new DateTime();

	/* Returns:
		{
			"firstname":"foo",
			"lastname":"bar",
			"birthdate": {
				"date":"2012-06-06 08:46:58",
				"timezone_type":3,
				"timezone":"Europe\/Berlin"
			}
		}
	*/
	json_encode($user);
	?>

Objects are inspected and their public attributes are converted. This happens recursively, so in the example above the public attributes of the [DateTime](http://at2.php.net/manual/en/datetime.construct.php) object are also translated into JSON. This is a handy trick if you want to easly transmit datetimes over JSON, since the client-side can then operate on both the actual time and the timezone.

	<?php
	class User {
		public $pub = "Mister X.";
		protected $pro = "hidden";
		private $priv = "hidden too";

		public $func;
		public $notUsed;

		public function __construct() {
			$this->func = function() {
				return "Foo";
			};
		}
	}

	$user = new User();

	// Returns: {"pub":"Mister X.","func":{},"notUsed":null}
	echo json_encode($user);
	?>

Here, you can see that only public attributes are used. Not initialized variables are translated to `null` while closures that are bound to a public attribute are encoded with an empty object (as of PHP 5.4, there is no option to prevent public closures to be translated).

### The $option bitmasks
Bitmasks are used to set certain flags on or off in a function call. This language pattern is commonly used in C and since PHP is written in C this concept made it up to some PHP function arguments as well. It's easy to use: if you want to set an option, just pass the constant as an argument. If you want to combine two or more options, combine them with the bitwise OR operation `|`. So, a call to json_encode may look like this:

	<?php
	// Returns: {"0":"Starsky & Hutch","1":123456}
	json_encode(array("Starsky & Hutch", "123456"), JSON_NUMERIC_CHECK | JSON_FORCE_OBJECT);
	?>

`JSON_FORCE_OBJECT` forces the array to be translated into an object and `JSON_NUMERIC_CHECK` converts string-formatted numbers to actual numbers. You can find all bitmasks (constants) [here](http://php.net/manual/en/json.constants.php). Note that most of the constants are available since PHP 5.3 and some of them were added in 5.4. Most of them deal with how to convert characters like `< >`, `&` or `""`. PHP 5.4 provides a `JSON_PRETTY_PRINT` constant that may you help during development since it uses whitespace to format the output (since it adds character overhead, I won't enable it in production of course).

## Decoding JSON in PHP
Decoding JSON is as simple as encoding it. PHP provides you a handy [json_decode](http://php.net/manual/de/function.json-decode.php) function that handles everything for you. If you just pass a valid JSON string into the method, you get an object of type `stdClass` back. Here's a short example:

	<?php
	$string = '{"foo": "bar", "cool": "attr"}';
	$result = json_decode($string);

	// Result: object(stdClass)#1 (2) { ["foo"]=> string(3) "bar" ["cool"]=> string(4) "attr" }
	var_dump($result);

	// Prints "bar"
	echo $result->foo;

	// Prints "attr"
	echo $result->cool;
	?>

If you want to get an associative array back instead, set the second parameter to true:

	<?php
	$string = '{"foo": "bar", "cool": "attr"}';
	$result = json_decode($string, true);

	// Result: array(2) { ["foo"]=> string(3) "bar" ["cool"]=> string(4) "attr" }
	var_dump($result);

	// Prints "bar"
	echo $result['foo'];

	// Prints "attr"
	echo $result['cool'];
	?>

If you expect a very large nested JSON document, you can limit the recursion depth to a certain level. The function will return `null` and stops parsing if the document is deeper than the given depth.

	<?php
	$string = '{"foo": {"bar": {"cool": "value"}}}';
	$result = json_decode($string, true, 2);

	// Result: null
	var_dump($result);
	?>

The last argument works the same as in json_encode, but there is only one bitmask supported currently (which allows you to convert bigints to strings and is only available from PHP 5.4 upwards).We've been working with valid JSON strings until now (aside fromt the `null` depth error). The next part shows you how to deal with errors.

## Error-Handling and Testing
If the JSON value could not be parsed or a nesting level deeper than the given (or default) depth is found, NULL is returned from json_decode. This means that no exception is raised by json_encode/json_deocde directly.

So how can we identify the cause of the error? The json_last_error function helps here. [json_last_error](http://at2.php.net/manual/en/function.json-last-error.php) returns an integer error code that can be one of the following constants (taken from [here](http://at2.php.net/manual/en/function.json-last-error.php)):

- JSON_ERROR_NONE: No error has occurred.
- JSON_ERROR_DEPTH: The maximum stack depth has been exceeded.
- JSON_ERROR_STATE_MISMATCH: Invalid or malformed JSON.
- JSON_ERROR_CTRL_CHAR: Control character error, possibly incorrectly encoded.
- JSON_ERROR_SYNTAX: Syntax error.
- JSON_ERROR_UTF8: Malformed UTF-8 characters, possibly incorrectly encoded (since PHP 5.3.3).

With those information at hand, we can write a quick parsing helper method that raises a descriptive exception when an error is found.

	<?php
	class JsonHandler {

		protected static $_messages = array(
			JSON_ERROR_NONE => 'No error has occurred',
			JSON_ERROR_DEPTH => 'The maximum stack depth has been exceeded',
			JSON_ERROR_STATE_MISMATCH => 'Invalid or malformed JSON',
			JSON_ERROR_CTRL_CHAR => 'Control character error, possibly incorrectly encoded',
			JSON_ERROR_SYNTAX => 'Syntax error',
			JSON_ERROR_UTF8 => 'Malformed UTF-8 characters, possibly incorrectly encoded'
		);

		public static function encode($value, $options = 0) {
			$result = json_encode($value, $options);

			if($result)  {
				return $result;
			}

			throw new RuntimeException(static::$_messages[json_last_error()]);
		}

		public static function decode($json, $assoc = false) {
			$result = json_decode($json, $assoc);

			if($result) {
				return $result;
			}

			throw new RuntimeException(static::$_messages[json_last_error()]);
		}

	}
	?>

We can now use the exception testing function [from the last post about exception handling](http://nitschinger.at/A-primer-on-PHP-exceptions) to test if our exception works correctly.

	// Returns "Correctly thrown"
	assertException("Syntax error", function() {
		$string = '{"foo": {"bar": {"cool": NONUMBER}}}';
		$result = JsonHandler::decode($string);
	});

Note that since PHP 5.3.3, there is a `JSON_ERROR_UTF8` error returned when an invalid UTF-8 character is found in the string. This is a strong indication that a different charset than UTF-8 is used. If the incoming string is not under your control, you can use the [utf8_encode](http://at2.php.net/manual/en/function.utf8-encode.php) function to convert it into utf8.

	<?php echo utf8_encode(json_encode($payload)); ?>

I've been using this in the past to convert data loaded from a legacy MSSQL database that didn't use UTF-8.

Summary
----------
JSON is a convenient, readable and easy to use data exchange format that seems to replace XML as the de-facto standard on the web. PHP has everything you need already built in and provides various configuration options if you use a recent version (> 5.3).

You should now be able to understand JSON, how to interact with it through PHP and how to handle possible errors. Happy encoding!

(By the way, you may also want to check out my primer about [Exceptions in PHP](http://nitschinger.at/A-primer-on-PHP-exceptions)!)
