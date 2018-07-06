+++
date = "2011-07-19"
draft = false
title = "Using Environments in Lithium"
description = "This post introduces you to Lithium Environments and also shows you how to tailor them to your needs."
tags = ["php","environments","lithium"]
slug = "Using-Environments-in-Lithium"
+++

## Introduction
Environments help you to manage multiple configurations for your application. Maybe you have a different database for testing than for production or you use file caching in development but [APC](http://www.php.net/manual/en/intro.apc.php) in production. If your framework does not support this (or a similar) concept, it can be a pain to code this overhead for yourself. Therefore, Lithium frees you from this by providing a `Environment` class (in the `\lithium\core` namespace) which handles everything for you automatically. Other concepts like `adapters` integrate with your environments and give you a high flexibility while maintaining a set of sensible defaults.

The next chapter is a whirlwind tour on how to use environments in many places in the framework. Once you know how to work with them, we'll see how you can change the behavior and adapt it further to the needs of your application.

## Using environments
Let's start with the most common use case of environments: define a separate database connection for `development`, `test` and `production`:

	Connections::add('default', array(
		'development' => array(
			'type' => 'MongoDb',
			'host' => 'localhost',
			'database' => 'blog_dev'
		), 
		'test' => array(
			'type' => 'MongoDb',
			'host' => 'localhost',
			'database' => 'blog_test'
		),
		'production' => array(
			'type' => 'MongoDb',
			'host' => 'localhost',
			'database' => 'blog_prod'
		)
	));

As you can see, we don't work with the `Environment` class here directly. Instead, the `Adaptable` class (from which the `Connections` class derives) takes care for us. To understand what's going on, investigate `_config()` method inside the `Adaptable` class.

	/**
	 * Gets an array of settings for the given named configuration in the current
	 * environment.
	 *
	 * The default types of settings for all adapters will contain keys for:
	 * `adapter` - The class name of the adapter
	 * `filters` - An array of filters to be applied to the adapter methods
	 *
	 * @see lithium\core\Environment
	 * @param string $name Named configuration.
	 * @return array Settings for the named configuration.
	 */
	protected static function _config($name) {
		//...
		$env = Environment::get();
		$config = isset($settings[$env]) ? $settings[$env] : $settings;
		//...
	}

The important part is done in the middle of the method. The `Environment::get()` method is called and based on the result it either uses the appropriate environment configuration or uses the whole array. This means that you are free to use environments or not.

You can use the `Environment::get()` method for yourself to check out the current environment. 

	use lithium\core\Environment;
	echo Environment::get();

For most of you, this will return `development` (if not, the next chapter will show you why). In addition to the `get()` method, there is also the `is()` method. This method takes a string as the param and returns either true or false if the environment matches your string. There are a lot of use cases for this, one can be found in your `routes.php` file:

	/**
	* Test routes.
	*/
	if (!Environment::is('production')) {
		Router::connect('/test/{:args}', array('controller' => 'lithium\test\Controller'));
		Router::connect('/test', array('controller' => 'lithium\test\Controller'));
	}

This code snippet only includes the `test` routes if you're not running a production setup. Here's another one:

	if(Environment::is('development')) {
		Libraries::add('li3_docs');
	}

This only includes the `li3_docs` library during development, because you generally won't need it in production or testing. And the last one:

	$default = array('adapter' => 'File', 'strategies' => array('Serializer'));

	if ($apcEnabled && Environment::is('production')) {
		$default = array('adapter' => 'Apc');
	}
	Cache::config(compact('default'));

This snippet tells Lithium to use file caching in development and APC for production (this is a modified version of the code you can find by default in `app/config/cache.php`). I won't give another example here, I think you get the point.

## Advanced usage
Now that we know how to use environments, let's see how we can tailor them to our needs. Most of the time you have either different environments or the default detection mechanism doesn't work as expected.

Before we start to modify the environment detection mechanism, we should look at how it is implemented by default:

	protected static function _detector() {
		return static::$_detector ?: function($request) {
			switch (true) {
				case (in_array($request->env('SERVER_ADDR'), array('::1', '127.0.0.1'))):
					return 'development';
				case (preg_match('/^test/', $request->env('HTTP_HOST'))):
					return 'test';
				default:
					return 'production';
			}
		};
	}

You can see that if the server address is "127.0.0.1" (which basically means you're running on localhost) then the environment defaults to "devlopment". If this isn't the case, it tries to find `test` in the `HTTP_HOST` settings. This is also used by the testing framework so you may want to keep this as it is. If none of these checks match, you're running in production mode.

Let's say we want to add a `staging` environment which is identified by its ip address `192.168.100.22`:

	use lithium\core\Environment;
	Environment::is(function($request) {
		switch (true) {
			case (in_array($request->env('SERVER_ADDR'), array('::1', '127.0.0.1'))):
				return 'development';
			case ($request->env('SERVER_ADDR') == '192.168.100.22'):
				return 'staging';
			case (preg_match('/^test/', $request->env('HTTP_HOST'))):
				return 'test';
			default:
				return 'production';
		}
	});

Easy enough, so here's a final example: assume we don't want to rely on this detection mechanisms. Instead, we set the `LITHIUM_ENVIRONMENT` variable in our apache config (maybe through `SetEnv LITHIUM_ENVIRONMENT "development"`) and use it in our application. If no config variable is set, let's default to `development`:

	use lithium\core\Environment;
	Environment::is(function($request) {
		return $request->env('lithium_environment') ?: 'development';
	});

Of course, you have to make sure that your custom environment routines get executed before the dependent parts get bootstrapped. You can either put it at the bottom of `app/config/boostrap/libraries.php` or add a new config file that gets loaded afterwards. 

## Wrapping up
After reading this post you should have a basic idea on how environments work in Lithium, how to use them and also how to tailor them to your needs. Keep in mind that they are also reused in other classes and patterns like `Adaptable`.