+++
date = "2010-12-16"
draft = false
title = "Understanding the Inflector"
description = "In this post you'll explore the Inflector and hopefully get a better understanding how it works and what it can do for you."
tags = ["php","lithium","inflector"]
slug = "Understanding-the-Inflector"
+++

### Introduction

The Inflector is one of many utility classes that ship with Lithium out of the box. Those classes are designed to assist you with common tasks that need to be done.

One of those tasks may be to pluralize, singularize, camel-case or humanize strings, database keys or other dynamic content. Let's see how the class itself is described:

> Utility for modifying format of words. Change singular to plural and vice versa. Under_score a CamelCased word and vice versa. Replace spaces and special characters. Create a human readable word from the others. Used when consistency in naming conventions must be enforced.

The Inflector can be found in `lithium\util` and you can get it into your namespace with `use lithium\util\Inflector`. The documentation can be found online at [lithify.me/docs](http://lithify.me/docs/lithium/util/Inflector).

### Methods

Here is a comprehensive list of the methods that the Inflector provides to you (the descriptions are quoted from their documentation):

+ __pluralize__: changes the form of a word from singular to plural.
+ __singularize__: changes the form of a word from plural to singular.
+ __camelize__: takes a under_scored word and turns it into a CamelCased or camelBack word.
+ __underscore__: takes a CamelCased version of a word and turns it into an under_scored one.
+ __slug__: returns a string with all spaces converted to given replacement and non word characters removed.
+ __humanize__: takes an under_scored version of a word and turns it into an human- readable form by replacing underscores with a space, and by upper casing the initial character.
+ __tableize__: takes a CamelCased class name and returns corresponding under_scored table name.
+ __classify__: takes a under_scored table name and returns corresponding class name.

There are more public methods in there, but we'll get to those later on. Now that you have an idea what the Inflector can do for you, we'll explore each of these methods in detail. One quick note: if you need more examples, the [tests](http://rad-dev.org/lithium/source/libraries/lithium/tests/cases/util/InflectorTest.php#highlight) are a great place to find some.

#### Inflector::pluralize()

The `pluralize()` method takes a singular word (string) and returns you the pluralized version of it. Here are some examples:

	Inflector::pluralize('Apple'); // returns "Apples"
	Inflector::pluralize('Menu'); // returns "Menus"
	Inflector::pluralize('News'); // returns "News"

If you are not sure what the pluralized word will look like, take a look at the `lithium\util\Inflector::$_plural` property.

#### Inflector::singularize()

The `singularize()` method takes a plural word (string) and returns you the singularized version of it. Here are some examples:

	Inflector::singularize('Houses'); // returns "House"
	Inflector::singularize('Bananas'); // returns "Banana"
	Inflector::singularize('Men'); // returns "Man"

If you are not sure what the pluralized word will look like, take a look at the `lithium\util\Inflector::$_singular` property.

#### Inflector::camelize()

You can hand the `camelize()` method either a slugged or a under_scored word and it will return you this word CamelCased or camelBacked (if you provide `false` as the second argument). Some examples:

	Inflector::camelize('foo_bar'); // returns "FooBar"	
	Inflector::camelize('foo_bar', false); // returns "fooBar"

#### Inflector::slug()

The `slug()` method takes a string and creates a "slugged" representation of it. This basically means that all spaces will be replaced with a given character (defaults to "-"), non-word characters will be removed and characters with accents like "ä" will be translated to their ASCII representation. Here are some examples:

	Inflector::slug('The truth - and- more- news'); // returns "The-truth-and-more-news"
	Inflector::slug('!@$#exciting stuff! - what !@-# was that?'); // returns "exciting-stuff-what-was-that"

#### Inflector::underscore()

The `underscore()` method basically takes a camel-cased word, slugs it and lowercases all characters.

	Inflector::underscore('TestField') // returns test_field
	Inflector::underscore('FeineÄpfel') // returns feine_aepfel

Note here that characters with accents will be transliterated, because of the translation into a slug. I don't know if this is the intended behavior so this may change in future releases.

#### Inflector::humanize()

The `humanize()` method takes an underscored word, removes a given seperator (defaults to "_") and uppercases the first characters of the words. Some examples from the core tests:

	Inflector::humanize('posts'); // returns "Posts"
	Inflector::humanize('posts_tags'); // returns "Posts Tags"
	Inflector::humanize('file_systems'); // returns  "File Systems"

#### Inflector::tableize()

The `tableize()` method takes your string, underscores (remember the slug?) and finally pluralizes it. As you can see, those methods use other low level methods provided to build exactly what you need. With this method, you can hand over a Model name and get the correct table name for it.

	Inflector::tableize('Post'); // returns "posts"
	Inflector::tableize('ArtistsGenre'); // returns "artists_genres"
	Inflector::tableize('FileSystem'); // returns "file_systems"

#### Inflector::classify()

The `classify()` method is basically the opposite to the `tabelize()` method. You hand it over a table name and get the correct class (Model) name back. It singularizes and camelizes the given string.

		Inflector::classify('artists_genres'); // returns "ArtistsGenre"
		Inflector::classify('file_systems'); // returns "FileSystem"
		Inflector::classify('news'); // returns "News"

### Add your own rules

If you need, you can add your own rules and/or override default rules. The `Inflector::rules()` method is responsible for that and works as a setter or getter for all stored rules. The core tests provide an example:

	Inflector::rules('singular', array('/rata/' => '\1ratus'));

You can also add transliterations that maps language specific or accented characters to ASCII ones (they are used to create slugs, for example). The core tests also provide a nice example:

	$this->assertNotEqual(Inflector::slug('JØRGEN'), 'JORGEN');
	Inflector::rules('transliteration', array('/Ø/' => 'O'));
	$this->assertEqual(Inflector::slug('JØRGEN'), 'JORGEN');

Good practice is to store your custom rules in a bootstrap file, so that they are immediately available to your application when it is fully loaded. If you need to do that on-the-fly, be sure to read the next chapter about caching translation results.

### Caching

You've probably heard or used the methods above previously. What you maybe don't know is, that the Inflector caches your `singularize`, `pluralize`,... calls and if you call the same method with the same word again it will be served directly out of an array.

That's pretty neat and doesn't do any harm when you properly define your custom rules in a bootstrap file. But you may run into issues when you call a translation method for word X, then add your own rules that affect X and then call the method again. It will be served straight out of the cache and your rules won't match. 

You can fix this by calling `Inflector::reset()` before you apply your rules (this will empty the caches). The next call will then apply your custom rules.

### Conclusion

The Inflector is a nice utility library that is used widely in the Lithium core and can easily be used by your application too. It's a fast and extensible way of dealing with dynamic content that needs to be pluralized, singularized, slugged and so on.

Finally, it does not have any external dependencies so you can even use the Inflector in your own libraries or frameworks.