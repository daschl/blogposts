+++
date = "2011-02-05"
draft = false
title = "Understanding the Lithium Router - Part 1"
description = "In this article series we'll take an in-depth look at the Lithium router. This first part bootstraps your knowledge and lays a foundation for more advanced topics."
tags = ["php","lithium","routing"]
slug = "Understanding-the-Lithium-Router-Part-1"
+++

### Introduction
The router is an integral part of the framework and has two main purposes. The first one is to match a URL against a set of previously connected routes, the second one is to generate a URL from a set of arguments (reverse routing). The router itself is very flexible as we'll see later on, but getting started is pretty easy. Lithium comes with a set of sensible default routes that help you to get on track immediately. While developing, you can always add, modify or remove routes and the application will instantly take these changes into account. When you are running in production mode, routes should be cached for performance reasons, because compiling and analyzing routes takes some time for every request. We'll see how this works in one of the upcoming posts.

At the time of writing, there is no full featured guide available. Nevertheless, Lithium provides an extensive API documentation that covers nearly every aspect of the routing system. For your convenience, here are some short links to [the Router](http://lithify.me/docs/lithium/net/http/Router) and [the Route](http://lithify.me/docs/lithium/net/http/Route) classes. Usually a good place for examples are unit tests, and the core developers worked hard to implement high code coverage and extensive unit tests. You can find them online [here](http://dev.lithify.me/lithium/source/libraries/lithium/tests/cases/net/http).

#### The Request/Response-Cycle
Before we dive into the router itself, it would be a good idea to take a look where the router is placed in the request/response-cycle and how it participates in the typical dispatching process.

The index.php-Page creates a new request object and hands it over to the dispatcher. The dispatcher hands this request object over to the router, who matches it against his connected routes and returns parameters that tell the dispatcher which controller and action to execute. If no matching route is found, an exception is raised. The dispatcher then instantiates the correct Controller object, which returns - after processing the associated models, controller actions and views - a response object. The dispatcher then sends the headers
and the content of the response object back to the browser.

After reading this, you may notice that the flow of the request and response objects is really straightforward. It is possible to modify or exchange every part of this dispatching process, so you have the full flexibility at your fingertips if you need it.

The whole Lithium routing functionality is organized in separate classes that perform distinct operations. Two classes mainly participate in the routing process. The `lithium\net\http\Router` has one or more  instances of `lithium\net\http\Route` connected to itself. When the router has to route the request, it iterates over the connected routes and checks if one of them match. The routes are checked in the same order as they're connected, so the first match wins the race. Therefore you need to be careful when designing your routes, as this may have some impact on your performance. When no matching routes are found (and no default routes are in place), then a `lithium\net\http\RoutingException` is raised. We'll use this behavior in the second part to render a 404-Page.

Enough talk, let's get started.

### Basics First
Usually, there is only one file you'll need to modify when you want to add or modify your application routes. The `app/config/routes.php` file is part of the bootstrap process and connects your routes to the router. If you open the file, you'll see that there are already a few routes defined. We'll look at them route by route, because they cover most of the syntax we'll usually need.

We always use the  `Router::connect()` method if we want to bind a given URL to a set of parameters. The first route is also called the "root", because this route will be used if the user hits your application with no additional parameters (like `http://www.example.com`).

    Router::connect('/', array('Pages::view', 'args' => array('home')));

The `Router::connect()` method expects at least one argument, two more can also be specified. We'll see more of them in later examples. The first argument is a string with the URL, the second one can be just a string or an array with information on how to map the URL to controllers and actions. For the first route, when the user hits the application with the root URL "/", the `app\controllers\PagesController` and its `view` action is called. Additionally, a "args" param with the value 'home' is passed to the controller. This argument is used afterwards in the controller action to render the `home` view. Of course this is not necessary, but it showcases the flexibility of routes.

The next route connects URLs like `/pages/foo` or `/pages/bar` to the PagesController and the view action.

    Router::connect('/pages/{:args}', 'Pages::view');

If you re-read the first route now, you should see obvious similarities. These two routes call the same controller and the same action, but the first route sets the `args` param to a static value, while the second route sets it dynamically (based on the request). You can also pass an array as the second argument, which can give you more flexibility. The equal method call with an array passed as the second argument would look like this:

	Router::connect('/pages/{:args}', array('controller' => 'pages', 'action' => 'view'));

This way of calling `Router::connect()`allows you to specify more arguments, but if you only need to route to a Controller/Action combination without further arguments, the first example is much shorter and as a result way easier to read.

The `{:args}` string acts as a placeholder, so let's have a look at the controller and see how he uses the passed arguments (`app\controllers\PagesController`).

	public function view() {
		$path = func_get_args();

		if (empty($path)) {
			$path = array('home');
		}
		$this->render(array('template' => join('/', $path)));
	}

The `args` keyword defined in the route is passed to the view action as a param. If no path was set, the action sets it to "home". Afterwards, the correct template is rendered. This is just one way to use the passed arguments, we'll see different methods in a minute.

The next code snippet connects testing routes, but only when we are not in the production environment. You see that the routing system is flexible enough to take other conditions of the environment into account.

	if (!Environment::is('production')) {
		Router::connect('/test/{:args}', array('controller' => 'lithium\test\Controller'));
		Router::connect('/test', array('controller' => 'lithium\test\Controller'));
	}

By default, Lithium searches for the controller in the `app\controllers` namespace, so if you want to specify a controller in a different path, you'll need to give Lithium the fully namespaced path (for example if you need to match a route against a controller in a plugin). These two routes match `/test` and also `/test/lithium/tests/cases/action/ControllerTest`.

The last routes you'll find in the `routes.php` file are the so called "default routes". Because of the fact that route matching occurs in the same order as they are connected, they need to be connected at last (otherwise these routes would always match every request).

	Router::connect('/{:controller}/{:action}/{:id:[0-9]+}.{:type}', array('id' => null));
	Router::connect('/{:controller}/{:action}/{:id:[0-9]+}');
	Router::connect('/{:controller}/{:action}/{:args}');

The `{:controller}`, `{:action}`and `{:type}` keywords are treated specially. These three keywords are used by the router to dynamically find and match the correct Controller/Action combination and render the result as the correct `lithium\net\http\Media`-type. If a optional `{:id}` is given, it is also passed down to the action. By convention, if no action is given `YourController::index()` is assumed. The default routes also make it easy to define a separate output type (like JSON or XML), which can be used to render the data in an other format instead of HTML. If you want to learn more about this feature, be sure to read my blog post about ["Adding an RSS Feed to your blog with Lithium"](http://nitschinger.at/Add-an-RSS-Feed-to-your-blog-with-Lithium).

These default routes match nearly everything, like `/posts`, `/posts/show/4` or `/posts/index.json`. This default routes work perfectly with ids generated by MySQL auto increment (or anything similar), but it doesn't work with MongoDB-IDs. Check out my blog post about ["Lithium Routes And MongoDB"](http://nitschinger.at/Lithium-Routes-And-Mongo-DB).

To understand the routing process full-stack, we also need to investigate how we can access the `Request`-object and its params. In the next snippet you'll find various examples of routes and how their params can be used in your controller. They always follow the same schema, so once you get used to it you should not need to look up the documentation again.

The slug-route is a bit more complex for now, but we'll cover these kind of routes in the next part.

	/**
	 * connects the root URL to the PostsController and the "index"-action.
	 */
	Router::connect('/', array('Posts::index'));

	/**
	 * connects login and logout shorthand URLs to the UsersController.
	 */
	Router::connect('/login', array('Users::login'));
	Router::connect('/logout', array('Users::logout'));

	/**
	 * connects slug URLs in my blog to the PostsController.
	 */
	Router::connect('/{:slug:[\w\-]+}', array('Posts::show'));

	/**
	 * the corresponding Posts::show() action (the slug is passed in the request object).
	 * the show-action can be called with an id (default route), or with a slug (previous route)
	 */
	public function show() {
		if(isset($this->request->id)) {
			$post = Post::first($this->request->id);
		} elseif(isset($this->request->params['slug'])) {
			$post = Post::first(array('conditions' => array('slug' => $this->request->params['slug'])));
		}
		if(!count($post)) {
			$this->redirect('Posts::index');
		}
		return compact('post');
	}

#### Generating URLs
As said previously, the routing infrastructure is not only capable of matching URLs to routes, but also matching a set of given parameters and generate a URL out of them. This technique is called "reverse routing". The HTML-Helper for example does this for if you use its ["link"](http://lithify.me/docs/lithium/template/helper/Html::link%28%29)-method. If you want to trigger the reverse routing functionality manually, you can do this by querying the `Router::match()` method. Here is an example from the Lithium documentation that illustrates this behavior. Both `Router::match()` queries return the URL "/login" as a result:

	Router::connect('/login', array('controller' => 'users', 'action' => 'login'));
	// params as an array
	$url = Router::match(array('controller' => 'users', 'action' => 'login'));
	// a more compact syntax
	$url = Router::match('User::login');

By using this method to generate URLs, you also make sure that your application is portable and works even if you place the application under a subdirectory on the server.

### Wrapping Up
After reading this post you should have a basic understanding on how the router works and how it participates in the whole request/response cycle. The next posts will cover more advanced topics like testing routes,
[REST](http://en.wikipedia.org/wiki/Representational_State_Transfer)ful routes, namespaces/prefixes, bypassing the framework altogether, catching exceptions and even various production deployment considerations.

If you want to read about a specific topic, feel free to comment below. Any feedback would be also greatly appreciated. Lastly, special thanks to [@nateabele](http://twitter.com/#!/nateabele) for reading through this post and providing important feedback.