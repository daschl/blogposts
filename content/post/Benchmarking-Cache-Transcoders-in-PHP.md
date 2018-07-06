+++
date = "2013-01-30"
draft = false
title = "Benchmarking Cache Transcoders in PHP"
description = "This post analyses and benchmarks classical object serialization, igbinary and JSON. It compares them and shows their benefits and shortcomings."
tags = ["php","cache","json","serialize"]
slug = "Benchmarking-Cache-Transcoders-in-PHP"
+++

## Motivation
Storing PHP objects (or simpler data types like arrays) in caches always requires some kind of transformation. You need a way of encoding/decoding data so that it can be stored and loaded properly. In most languages, this process is known as object [serialization](http://en.wikipedia.org/wiki/Serialization). PHP provides a mechanism for this out of the box, but in this article we'll also look at [igbinary](http://pecl.php.net/package/igbinary) as a drop-in replacement for the default serializer. We also compare the results to object transcoding based on [JSON](http://json.org/), which is not really an object serialization mechanism but commonly used as a data chache structure which has its own benefits and drawbacks.

As always, please take this benchmarks with a grain of salt. Don't take the absolute numbers as a reference, look at the differences and run the benchmarks in your environment to get accurate results that apply to your scenarios. If you spot any flaws in what is shown here, please point it out in the comments. This blog post is not some kind of "advertising" for a special mechanism and all different transcoders are discussed with their benefits and drawbacks.

If you want to compare your results to mine, I'm using a MacBook Pro on Mac OS X 10.8.2 with the 2.3 GHz i7 and 16GB RAM. I'm using PHP 5.4.11 (through [homebrew-php](https://github.com/josegonzalez/homebrew-php)) instead of the shipped 5.3.

## The PHP Serializer
Out of the box, PHP provides a serialization mechanism on top of the [serialize](http://php.net/manual/de/function.serialize.php) and [unserialize](http://www.php.net/manual/de/function.unserialize.php) methods. By applying this method on a variable, it gets encoded to a byte stream and neither the type nor the structure gets lost (you can't store [resources](http://www.php.net/manual/en/language.types.resource.php)).

So, let's look at a simple and a more complex example on how the transformation looks like:

	<?php

	class Person {
		private $name;

		public function __construct($name) {
			$this->name = $name;
		}

		public function getName() {
			return $this->name;
		}
	}

	$simple = array("key" => "value");
	$complex = new Person("Michael");

	?>

If we're calling `serialize()` on both variables, the output (as a string representation) looks like this:

	<?php
	$simpleSerialized = serialize($simple);
	$complexSerialized = serialize($complex);

	// string(28) "a:1:{s:3:"key";s:5:"value";}"
	var_dump(serialize($simple));

	// string(51) "O:6:"Person":1:{s:12:"\000Person\000name";s:7:"Michael";}"
	var_dump(serialize($complex));
	?>

While the resulting string is not designed for readability, you can clearly see that some metadata is added in order to make the unserialization possible. For example, the `key` string is serialized to `s:3:"key"` where the `s` means `string` and `3` is the string length. Also, `a` points to an `array` and `O` to an `object`.

We can now `unserialize()` the values and work with them as if they've never been stored somewhere else:

	<?php
	$simpleUnserialized = unserialize($simpleSerialized);
	$complexUnserialized = unserialize($complexSerialized);

	// array(1) {'key' => string(5) "value"}
	var_dump($simpleUnserialized);

	// string(7) "Michael"
	var_dump($complexUnserialized->getName());
	?>

In practice, two characteristics are important: size of the serialized value and performance. While from a developer perspective performance is fun to explore, the size of the serialized object is what matters most. Performance differences are only measurable when running it in a loop with lots of iterations, while in practice you may only work with a few objects at a time per request.

Let's use a common caching scenario: imagine we're caching entity responses from an ORM framework like [Doctrine](http://www.doctrine-project.org/). Here is a blog post class that will be filled with life:

	<?php
	class BlogPost {

		private $title;
		private $teaser;
		private $author;
		private $body;

		public function __construct($title, $teaser, $author, $body) {
			$this->title = $title;
			$this->teaser = $teaser;
			$this->author = $author;
			$this->body = $body;
		}

		public function getTitle() { return $this->title; }
		public function getTeaser() { return $this->teaser; }
		public function getAuthor() { return $this->author; }
		public function getBody() { return $this->body; }

	}
	?>

Please take this with a grain of salt, because since JSON transcoding is not able to store a 1:1 representation like `serialize`, we need to work around that a bit. More details and discussion will be provided in the JSON section.

Assume that a blog post with content has around 10k characters and about 10kb in size, lets create a crafted blog post:

	<?php
	$randomString = function ($length) {
		$chars = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789 .";	
		$str = "";
		for( $i = 0; $i < $length; $i++ ) {
			$str .= $chars[rand(0, strlen($chars)-1)];
		}
		return $str;
	};

	$title = "How to measure caching transcoders";
	$teaser = "This is a much longer introduction on how to measure caching transcoders. Feel free to post your own findings.";
	$author = "daschl";
	$body = $randomString(10000);

	$post = new BlogPost($title, $teaser, $author, $body);
	?>

Now run `serialize` and `unserialize` in loops to measure performance as well as determine the resulting size of the object:

	<?php
	$iterations = 10000;
	$start = microtime(true);
	for($i=0;$i<$iterations;$i++) {
		$serialized = serialize($post);
	}
	$end = microtime(true);

	echo "Size: " . strlen($serialized) . "\n";
	echo ($end - $start) * 1000 . "Milliseconds\n";
	?>

On my machine, to serialize 10000 objects it took about 33 milliseconds. The length of the resulting object (its string representation) is 10297 characters. This is kind of expected because the bulk of the object is the long article body. Before start comparing, we can already make one observation: its pretty fast. Now, let's move on to igbinary serialization and see how these results compare.

## Serializing with igbinary
The [igbinary](http://pecl.php.net/package/igbinary) extension was developed as a drop-in replacement for the default PHP serializer. Instead of storing the serialized object as a text, it is stored as binary data. You can install it either through PECL or compile it from source, either way before running the tests make sure you have the `igbinary` extension enabled:

	michael@daschlbook ~ $ php -m | grep igbinary
	igbinary

Instead of using `serialize` or `unserialize`, igbinary provides us with appropriate `igbinary_serialize` and `igbinary_unserialize` functions. Now we can apply the same microbenchmark and see the results:

	<?php
	$iterations = 10000;
	$start = microtime(true);
	for($i=0;$i<$iterations;$i++) {
		$serialized = igbinary_serialize($post);
	}
	$end = microtime(true);

	echo "Size: " . strlen($serialized) . "\n";
	echo ($end - $start) * 1000 . "Milliseconds\n";
	?>

With default settings, to serialize 10000 objects with igbinary it took 678 milliseconds. The length of the resulting object is 10244 characters. If you compare it with the default serialization mechanism, that is 20 times slower and the resulting object isn't much smaller either. Bummer - or not? Let's look at why this is happening.

Let's look at a different workload. Let's create a data structure which consist of 30 smaller blog posts and measure the statistics for both again (only doing 1000 iterations instead of 10000):

	<?php
	$posts = array();
	for($i=0;$i<30;$i++) {
		$posts[] = new BlogPost($title, $teaser, $author, $randomString(50));
	}
	?>

The results are more interesting: on my machine igbinary takes about 95 milliseconds while serialize needs only 41 milliseconds. Now it's just 2 times slower. But here comes the important part: the resulting size of igbinary is 2385 characters, while serialize is 10467 characters! That's about 5 times smaller. Note that this gap increases the larger your PHP object gets (not in terms of long character strings but speaking of complexity like methods).

Note that the igbinary docs say you should set `igbinary.compact_strings` to `Off` to increase performance, but there was no real measurable difference in my testings.

There is one more thing we haven't compared: deserializing of previously serialized objects. We can take the same code from above, use the serialized part as input and measure the timings once again (not that we don't need to measure the size of the resulting object because we're creating a "live" object again):

	<?php
	$posts = array();
	for($i=0;$i<30;$i++) {
		$posts[] = new BlogPost($title, $teaser, $author, $randomString(50));
	}

	$serializedIgbinary = igbinary_serialize($posts);
	$serializedClassic = serialize($posts);

	$iterations = 1000;
	$start = microtime(true);
	for($i=0;$i<$iterations;$i++) {
		$unserialized = igbinary_unserialize($serializedIgbinary);
	}
	$end = microtime(true);

	echo ($end - $start) * 1000 . "Milliseconds\n";

	$start = microtime(true);
	for($i=0;$i<$iterations;$i++) {
		$unserialized = unserialize($serializedClassic);
	}
	$end = microtime(true);

	echo ($end - $start) * 1000 . "Milliseconds\n";
	?>

This is giving us very interesting results: igbinary takes 65 milliseconds while classic unserialize needs 82 milliseconds! So igbinary is faster here. This is pretty good news, because in practice you will need to unserialize (speak "fetch from cache") much more often than to serialize (speak "store in cache"). That's the whole point of a cache.

Before we move on to JSON, I think it's fair to say that igbinary wins this comparison. Most of the time you want to store complex PHP objects and fetch them out fast and quickly. Of course it has the overhead of installing it as an extension, but since it's a drop-in replacement you can change your mind later (but take this with a grain of salt because you can only unserialize what was serialized with igbinary and vice versa - so you'd need to fetch and restore them).

Finally, when you are storing very long strings in objects, both have pretty much the same overhead (because you can't "compact" this long string very well). One workaround may be to compress and decompress the string as needed, using a mixture of gzip and base64 encoding. When serializing PHP objects, you can use the [__sleep](http://www.php.net/manual/en/language.oop5.magic.php#object.sleep) and [__wakeup](http://www.php.net/manual/en/language.oop5.magic.php#object.wakeup) methods to perform these kind of transformations.

Adding these two methods to our `BlogPost` class results in a 20% saving when using the 10k random string again:

	<?php
	public function __sleep() {
		$this->body = gzdeflate($this->body, 9);
		return array('title', 'teaser', 'author', 'body');
	}

	public function __wakeup() {
		$this->body = gzinflate($this->body);
	}
	?>

Note that compressing takes time too, so measure if it makes sense in your case and the space savings are good enough. You also need the `zblib` extension to make this work properly.

## Transcoding on top of JSON
First of all, JSON has not the same characteristics as serialization. This is just because JSON only supports a subset of the available data types that PHP supports and therefore encoding/decoding takes some extra steps on the application side. So why are we comparing it here? Well, JSON is used very often for storing data in caches. Not only is it human readable, it can also be used in combination with other languages and applications. Using JSON allows you to store cache information from a Java backend and use it in your PHP application. That doesn't work with traditional serialization approaches.

If you need a primer on JSON and PHP, [read](http://nitschinger.at/Handling-JSON-like-a-boss-in-PHP) my other article first. PHP allows you to transcode all objects (except resources) through the [json_encode](http://www.php.net/manual/en/function.json-encode.php) and [json_decode](http://www.php.net/manual/en/function.json-decode.php) functions. Private and protected variables are not converted, so we either need to make them public or provide a custom method that exports a storable object structure. While we're at it, we can provide a static factory method that initializes our BlogPost from the returned JSON string:

	<?php
	public function toJson() {
		return array(
			'title' => $this->title,
			'teaser' => $this->teaser,
			'author' => $this->author,
			'body' => $this->body
		);
	}

	public static function fromJson($json) {
		$d = json_decode($json, true);
		return new BlogPost($d['title'], $d['teaser'], $d['author'], $d['body']);
	}
	?>

Note that the `toJson()` method returns a simple representation of the object, which is much less complicated than a full serialization approach. Let's run our benchmark again:

	<?php
	$post = new BlogPost($title, $teaser, $author, $randomString(10000));
	$post = $post->toJson();

	$iterations = 10000;
	$start = microtime(true);
	for($i=0;$i<$iterations;$i++) {
		$encoded = json_encode($post);
	}
	echo "Size: " . strlen($serialized) . "\n";
	$end = microtime(true);

	echo ($end - $start) * 1000 . "Milliseconds\n";
	?>

On my machine it takes 1.7 seconds (!) to complete the encoding. The resulting object size is 10196 bytes which is comparable to our earlier measurements (again, the large value dominates here). Testing it against complex objects is not interesting because we can't restore them anyway without further modifications. Running `json_decode` on the encoded object also takes around 1.5 seconds, so there is no major difference here.

Personally, I'd expected to have JSON transcoding to perform better, but again normally you don't run 10k iterations in a row. When you break it down to 10 encodings they take about 1 millisecond which is okay for most scenarios (given we encode a really large objects with a few kb here). Smaller objects like user sessions will encode much faster.

Edit: a friendly reader on [reddit](http://www.reddit.com/r/PHP/comments/17kcy3/benchmarking_cache_transcoders_in_php/) pointed out that you can (and should) use the [JsonSerializable interface](http://www.php.net/manual/en/class.jsonserializable.php) instead of rolling your own method name like `toJson()`. Thanks!

Edit2: another reader pointed out that naming a method `toJson()` which does not return JSON but an array is misleading. I agree, so if you adapt this approach in your environment choose a better name like `prepareForJson()`, `toArray()` or use the `JsonSerializable` Interface as shown above.

## Conclusion
After running these microbenchmarks, my conclusion is that when you need object serialization, go with igbinary. It provides good enough serialization performance and huge wins on object sizes. Decoding performance is also much better than out-ouf-the-box serialization. If you need interoperability with other applications or if you don't want to limit yourself to serialized PHP blobs, go with JSON. JSON requires much more hand wiring but you'll gain a lot of flexibility.