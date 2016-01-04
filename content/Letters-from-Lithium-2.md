+++
date = "2011-09-05"
draft = false
title = "Letters from Lithium #2"
description = "Letters from Lithium keeps you up to date with Lithium and its community. Issue #2 2011-09-05"
tags = ["php","lfl","lithium"]
slug = "Letters-from-Lithium-2"
+++

## About
Letters from Lithium keeps you up to date about what happens in the Lithium core and community. If you have interesting news to share, comment below or ping me on [twitter](http://twitter.com/daschl).

## Core News
Routing got a bit more awesome with [route continuations](https://github.com/UnionOfRAD/lithium/commit/792b916249052b87c368e2e25798a93dceaa0154#L3R636). They allow you match more routes in one request, which is very useful in many cases. Take a look at this test case:

	/**
	* Tests that continuation routes properly fall through and aggregate multiple route parameters.
	*/
	public function testRouteContinuations() {
		Router::connect('/{:locale:en|de|it|jp}/{:args}', array(), array('continue' => true));
		Router::connect('/{:controller}/{:action}/{:id:[0-9]+}');

		$request = new Request(array('url' => '/en/posts/view/1138'));
		$result = Router::process($request)->params;
		$expected = array (
			'controller' => 'posts', 'action' => 'view', 'id' => '1138', 'locale' => 'en'
		);
		$this->assertEqual($expected, $result);
	}

The data branch has been [merged](https://github.com/UnionOfRAD/lithium/commit/107f596b3224b47cfe91a820bae5867a5e0aee6c) recently, which should improve the overall stability of the model layer and add a few smaller features. Thanks to [Veselin Todorov](https://github.com/vesln), you [can now use](https://github.com/UnionOfRAD/lithium/commit/0c35bef1a36f3ffd6baee3586bf8c3380dfabb81) the "last" option in validations. This lets you better control the validation stack.

[Richard McIntyre](https://github.com/mackstar) added [console environment detection](https://github.com/UnionOfRAD/lithium/commit/427ee643e0725cceb42d6f1ee66cf5668d18effa) and Nate Abele improved the application detection process.

Of course, various other bugs have been fixed (including the [HMAC bug](https://github.com/UnionOfRAD/lithium/commit/cbd0c4d2c07837e2e5ea4f9e8ff8c6a0d1fa1c61)) and small enhancements like [collection sorting](https://github.com/UnionOfRAD/lithium/commit/74514b870c13aac98236111c524ab5c1bc04656f) and [Form::error() filtering](https://github.com/UnionOfRAD/lithium/commit/c0d4ff82b1b3bb662ed0e78ed9599437306ca068) found their way into the core.

## Plugins
[eLod](https://github.com/eLod) (also known as "smoking_kiddo" on IRC) created a plugin called [li3_console](https://github.com/eLod/li3_console) which provides a fully functional PHP [REPL](http://en.wikipedia.org/wiki/Read-eval-print_loop) and also lets you interact with your Lithium environment (like models) from the command line.

If you need to deploy your Lithium application to remote servers and are used to [Capistrano](https://github.com/capistrano/capistrano), you should take a look at [capium](https://github.com/mehlah/capium). It is a ruby gem that provides deploy recipes and rake tasks for Lithium.

## Blogs and Media
For those new to Lithium, [David Golding](http://www.davidgolding.net/) recently published a very informative introduction [here](http://www.scribd.com/doc/62146078/New-to-Lithium-Part-1). It gets you up and running quickly and walks you through the creation of models, controllers and views.

Also, [James Fuller](http://www.jblotus.com/) released a few blog posts in the last weeks, covering various topics around Lithium. Head over to his blog and read about [Flash Messages](http://www.jblotus.com/2011/08/17/adding-a-session-flash-message-to-your-site-in-lithium-php/), the [FormHelper](http://www.jblotus.com/2011/08/19/how-to-add-a-html-button-element-using-lithium-phps-formhelper/), [Web Service Calls](http://www.jblotus.com/2011/08/20/super-simple-lithium-php-json-web-service-calls/), [Filters](http://www.jblotus.com/2011/08/27/understanding-filters-in-lithium-php/) and [AppControllers](http://www.jblotus.com/2011/08/31/using-an-appcontroller-in-lithium-php-to-pass-data-to-the-layout/).

## Universe
Recently, [Engine Yard](http://www.engineyard.com/) (mostly known for their knowledge and cloud platforms in the Ruby on Rails world) acquired [Orchestra](http://orchestra.io) to extend their language support. Orchestra is a great platform to deploy your PHP and Lithium applications on to, so give it a try (they also provide free accounts) if you haven't already. The [manual](http://lithify.me/docs/manual/cloud/readme.wiki) also provides a short guide if you need help.

[Ciaro Vermeire](https://github.com/Ciaro) started a (unofficial) community forum for Lithium, which you can find [here](http://lithium-framework.com/). If you like this type of communication, feel free to register, ask for help and - most importantly - share what you know.