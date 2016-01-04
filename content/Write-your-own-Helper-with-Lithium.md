+++
date = "2010-11-23"
draft = false
title = "Write your own Helper with Lithium"
description = "In this post I'll show you how to create your own View-Helper with Lithium. You'll also learn something about the Inflector and Unit-Testing."
tags = ["php","lithium","helper","inflector"]
slug = "Write-your-own-Helper-with-Lithium"
+++

Lithium comes with two helpers out of the box. The Form-Helper and the Html-Helper. Both are great and help you to make your View-Code more flexible, maintainable and easier to read.

But what do you do when you need a functionality that they don't provide? Right, you write your own. Let's imagine you want to tell your blog-readers how many posts you have stored in your database. In our first iteration our code might look like this:

    We have <?= count($posts) ?> posts stored in the database.

Looks great so far, but our code doesn't cover the case when we have only one post stored. Let's fix that:

    We have <?= count($posts) ?> 
    <?php if(count($posts) != 1): ?>
        posts
    <?php else: ?>
        post
    <?php endif; ?>
    stored in the database.

This works, but looks ugly and makes our View more complicated and messy (leaving aside that the above code is a bit verbose). To fix that, we write a custom `Word-Helper` that abstracts the count-stuff for our view.

Thankfully, the li3-developers thought about that and provide us with everything we need to get started. Lithium also automatically loads our helper if we want to access it. Let's assume that we want to call the helper with `$this->word->pluralize(count($posts), 'post')`.

We all love tests, so let's start with them first. We create a new `WordTest.php` (in `app/tests/cases/extensions/helper`) and add a some test cases. 

	<?php
	namespace app\tests\cases\extensions\helper;
	use app\extensions\helper\Word;
	
	class WordTest extends \lithium\test\Unit {
		public function setUp() {
			$this->word = new Word();
		}

		public function testPluralize() {
			$singular = 'post';
			
			$expected = '1 post';
			$this->assertEqual($expected, $this->word->pluralize(1, $singular));
			
			$expected = '0 posts';
			$this->assertEqual($expected, $this->word->pluralize(0, $singular));
			
			$expected = '2 posts';
			$this->assertEqual($expected, $this->word->pluralize(2, $singular));
		}
	}
	?>

If we run the tests (`http://yourblog.com/test/app/tests/cases/extensions/helper/WordTest`), they will fail (as expected). In the `setUp`-method we try to instantiate the Word-Helper, but it doesn't exist yet. The `testPluralize`-method tests three different calls of our `Word::pluralize()` Helper-Method. To make our tests go green, we finally create our `Word`-Helper (in `app/extensions/helper/Word.php`).

	<?php
	namespace app\extensions\helper;
	use lithium\util\Inflector;

	class Word extends \lithium\template\Helper {
		public function pluralize($count, $word) {		
			$word = ($count != 1 ? Inflector::pluralize($word) : $word);
			return $count." ".$word;
		}
	}
	?>

Now re-run your tests - they should succeed. If they don't, please check that your files are named correctly and you don't have any typos in them.

Now it's time to call our brand-new helper from our view. Thanks to the `autoloading` feature, it's dead easy:

    We have <?= $this->word->pluralize(count($posts), 'post'); ?> stored in the database.

That's it - you've just created your own View-Helper! If you have any questions or suggestions, feel free to comment below.