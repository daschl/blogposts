+++
date = "2010-12-04"
draft = false
title = "Introducing Fixtures for Lithium"
description = "With the new li3_fixtures plugin you can simplify your tests, DRY them up and make them more readable."
tags = ["php","fixtures","lithium","plugin"]
slug = "Introducing-Fixtures-for-Lithium"
+++

When you write tests for your classes (and you should), you may run into the problem that you create large arrays of test data in your code. Consider the following example:

	$expected = array(
		'post1' => array(
			'title' => 'My First Post',
			'content' => 'First Content...'
		),
		'post2' => array(
			'title' => 'My Second Post',
			'content' => 'Also some foobar text'
		),
		'post3' => array(
			'title' => 'My Third Post',
			'content' => 'I like to write some foobar foo too'
		)
	);

	$this->assertEqual($expected[0], Post::first());
    /* more tests down here */

This creates a nested array of test data where each inner array mocks a `post` stored in the database. If we have to do this more than once (maybe in an other test we insert this data into the database to test validations), we may put the snippet in the `setUp()` method. A better approach would be to move this code in fixture files. The `li3_fixtures` plugin tries to help you with that and provides a simple and convenient approach to integrate fixtures in your tests.

Before we can use the plugin, we need to install it first. `li3_fixtures` doesn't have any dependencies, so this is straightforward. Note that for now you need a rad-dev.org-account to clone the repository.

    $ cd /path/to/app/libraries
    $ git clone code@rad-dev.org:li3_fixtures.git
    $ mkdir /path/to/app/tests/fixtures

Now we need to tell our app to use the plugin, so fire up your editor and modify your `app/config/bootstrap/libraries.php`.

    Libraries::add('li3_fixtures');

After that, we can create a sample fixture file in `app/tests/fixtures/posts.json`. Currently only json-files are supported, but xml-support is planned.

	{
		"post1": {
			"title": "My First Post",
			"body": "First Content..."
		},
		"post2": {
			"title": "My Second Post",
			"body": "Also some foobar text"
		},
		{
			"title": "My Third Post",
			"body": "I like to write some foobar foo too"
		}
	}

Now that we have our fixture file in place, lets see how the `$this->assertEqual()` looks like (don't forget to use the `li3_fixtures` plugin with `use li3_fixtures\test\Fixture;`):

	$fixtures = Fixture::load('Post');
	$this->assertEqual($fixtures->first(), Post::first());

By convention, your `$model` in `Fixture::load($model)`well be lowercased and pluralized by the Inflector (so 'Post' loads 'posts.json'). The `load`-method also takes an optional argument where you can override various default settings like the extension or location of the fixture-file.

As you can see, your test code is significantly smaller and easier to read. You may also notice the `first()`-method here. `li3_fixtures` makes use of the built in `lithium\util\Collection` class, which provides a powerful set of collection management methods like `first()`, `next()` or `prev()`. Check out the `lithium\util\Collection` documentation for more info on that.

The project can be found at [rad-dev.org/li3_fixtures](http://rad-dev.org/li3_fixtures) and is maintained by me, so if you have any comments or questions, feel free to ask comment here or contact me directly.