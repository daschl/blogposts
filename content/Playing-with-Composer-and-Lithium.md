+++
date = "2012-01-23"
draft = false
title = "Playing with Composer and Lithium"
description = "This post shows you how to manage your dependencies with Composer."
tags = ["php","lithium","composer","packagist"]
slug = "Playing-with-Composer-and-Lithium"
+++

## About Composer
[Composer](http://packagist.org/about-composer) is a command-line tool that helps you manage your application dependencies. It automatically 
fetches packages, resolves dependencies and is easy to configure. The really good thing about Composer is that it isn't bound to a specific framework and can be used with every kind of repository. Composer is similar to package managers like npm, so you may feel at home quickly.

The default repository of Composer is [Packagist](http://packagist.org/). If you want to use Composer in your project, you basically need two things: a `composer.json` file in your application root and the `composer.phar` application file itself. The easiest way to get it is to download it [directly](http://getcomposer.org/composer.phar) and drop it somewhere on your file system. A minimal `composer.json` file looks like this:

	{
		"name": "my-project",
		"version": "1.0.0",
		"require": {
			"monolog/monolog": "1.0.0"
		}
	}

As Packagist acts as the default repository, this will fetch the [monolog](http://packagist.org/packages/monolog/monolog) library from there into the `vendor` directory. There are a lot of configuration options that can be found [here](https://raw.github.com/composer/composer/master/doc/composer-schema.json). 

Let's go through a quick example hands-on. Consider the following directory structure:

	/web
		composer.phar
		/my-project
			composer.json

If you execute the following commands:

	$ cd my-project/
	$ php path/to/composer.phar install

Your directory structure will look like this afterwards:

	/web
		composer.phar
		composer.lock
		/my-project
			composer.json
			/vendor
				/bin
				/monolog
					....

If you stick with the defaults, all your dependencies will be installed in the `vendor` directory. Lithium uses the `libraries` directory to store the dependencies instead, but composer makes it easy to change the default directory (as you'll see shortly). You may also notice that there's a `bin` directory, which contains executable files (of course only if the installed  dependencies provide some).

Composer also creates a `composer.lock` file that contains a "frozen" state of the current dependency tree.

	{
		"hash": "a725fb1bf93f5c534217bbce2897ddc9",
		"packages": [
			{
				"package": "monolog\/monolog",
				"version": "1.0.0"
			}
		]
	}

## Managing Lithium
Now that we know how to work with Composer, let's manage a Lithium application with it. Currently, Lithium doesn't provide Composer packages out of the box, but it's easy to write one.

The first thing you want to do is clone (or download) the `framework` repository. You can also use the `li3 library extract` command but then you'd have to change the `LITHIUM_LIBRARY_PATH` back to the default location.

	$ git clone git://github.com/UnionOfRAD/framework.git composer-test
	Cloning into composer-test...
	remote: Counting objects: 29794, done.
	remote: Compressing objects: 100% (8622/8622), done.
	remote: Total 29794 (delta 18425), reused 29672 (delta 18332)
	Receiving objects: 100% (29794/29794), 3.94 MiB | 1.52 MiB/s, done.
	Resolving deltas: 100% (18425/18425), done.

Now, instead of initializing the git submodules (which would fetch the Lithium core the good old way), add this `composer.json` file to `/composer-test`.

	{
		"name": "composer-test",
		"version": "0.1.0",
		"config": {
			"vendor-dir": "libraries"
		},
		
		
		"repositories": {
			"UnionOfRAD": {
				"package": {
					"name": "lithium",
					"version": "master",
					"source": {
						"url": "git://github.com/UnionOfRAD/lithium.git",
						"type": "git",
						"reference": "master"
					}
				}
			}
		},
		
		"require": {
			"lithium": "master"
		}
	}

The `vendor-dir` setting changes the default `vendor` directory to `libraries`, which is recognized automatically by the Lithium class loader. If you don't like this approach, you could also change the `LITHIUM_LIBRARY_PATH` and point it to the `vendor` directory or create a symlink, but we stick with it for now. The `repositories` setting 
adds the git repository of the lithium core (if Lithium would provide a package on Packagist, then you wouldn't have to do this). The `require` setting is where all your application dependencies go into. The key `lithium` here refers to the package name in the `repositories` setting above.

Let's install the dependencies:

	$ php path/to/composer.phar install
	Installing dependencies
	Writing lock file
	Generating autoload files

If you look into the `/composer-test/libraries` directory, you can see that the `lithium` directory has been added successfully.

If we now want to install [twig](http://twig.sensiolabs.org/), we can modify our `composer.json` file accordingly:

	"require": {
		"lithium": "master",
		"twig/extensions": "master-dev"
	}

Not that the twig library is already available on Packagist, so we don't have to tell Composer where to find it. We can now run `composer.phar update` to update our dependencies (and the `composer.lock` file):

	$ php ../composer.phar update
	Updating dependencies
	- Package twig/twig (1.6.0-dev)
		Downloading
		Unpacking archive
		Cleaning up

	- Package twig/extensions (master-dev)
		Downloading
		Unpacking archive
		Cleaning up

	Writing lock file
	Generating autoload files

As you can see, Composer has automatically downloaded all dependencies needed by `twig/extensions` too!

You can now start managing your dependencies through composer, regardless if they actually provide composer packages or not. There is an [ongoing discussion](https://github.com/UnionOfRAD/lithium/issues/285) on how Lithium will handle dependency management in the future, so stay tuned.
