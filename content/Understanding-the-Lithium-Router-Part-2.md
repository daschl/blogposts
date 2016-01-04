+++
date = "2011-02-24"
draft = false
title = "Understanding the Lithium Router - Part 2"
description = "In this part we will look at testing your routes. This post covers unit testing, integration testing and reverse routing tests."
tags = ["php","lithium","routing","testing"]
slug = "Understanding-the-Lithium-Router-Part-2"
+++

### Introduction
Routes play an essential role in your request/response-cycle and therefore should also be tested like any other component that you develop. As the Lithium routing infrastructure also consists of classes and methods, we can run unit and integration tests against them.

If we follow the testing conventions, we need to differentiate two distinct methods of testing. The first one (the so called "[Unit Test](http://en.wikipedia.org/wiki/Unit_testing)"), is used to test your routes one by one, isolated from your application and ideally all dependencies are mocked away. The second one (called "[Integration Test](http://en.wikipedia.org/wiki/Integration_testing)"), takes the state of your actual application into account and therefore tests the routes in a "real environment". This way we can validate their functionality in isolation, and when we are confident with that we can check that no dependencies or other interfaces change the expected behavior.

Let's cover unit tests first and then move on to the integration tests. Afterwards, we'll look at testing reverse routes (more on that later).

### Unit Testing
Let me clarify one thing first: in general, you don't need to unit test your routes (instead you only need to do integration testing, but more on that later). There core developers did an awesome job in unit testing the routing infrastructure for you, so there is no need for you to duplicate their tests.

Nevertheless, learning  something about unit testing routes has two benefits. On the one hand, you get some practice in writing unit tests and on the other hand you get a better understand on how the routing system works internally. So i really recommend you to read through this section and play around with the code.

If we don't know how to implement unit tests, it is always a good idea to look at the core tests, as they show us how to do this properly and at the same time give us use cases that we can use in our own application. We can also copy and modify some tests, because often our own tests will be similar.

Before we can unit test some routes, we need to add a unit test file. According to the namespace standard, let's create the appropriate file (`RouteTest.php`) in `app\tests\cases\net\http`:

	<?php
	namespace app\tests\cases\net\http;

	use lithium\action\Request;
	use lithium\net\http\Route;

	class RouteTest extends \lithium\test\Unit {

	}

	?>

In our first test, we want to make sure that the route `/foo`, matches the `FooController` and the `index` action. Insert the following test case in your previously created class and run it through the test runner ("http://example.com/test/app/tests/cases/net/http/RouteTest").

	public function testFooRoute() {
		$params = array('controller' => 'foo', 'action' => 'index');
		$route = new Route(array(
			'template' => '/foo',
			'params' => $params
		));

		$request = new Request();
		$request->url = '/foo';

		$result = $route->parse($request);
		$this->assertEqual($params, $result->params);
	}

If we run `Router::connect()`, the `Router` basically creates a new `Route` object and passes the arguments to its constructor. In our tests we do the same, but we initialize the Route directly. This way, we can test the route in isolation. When we create a new route, the `template` param is the URL that our route will match (remember that it can also contain non-static identifiers). The params argument tells
the route what to return when the template matches the request. We then create a simple `Request` object (that mimics a request by the browser) and hard-code a URL (in the real application, the `$request->url` will be dynamically assigned). Now we run `parse()`,
that's exactly what the Router does route for route when the `Dispatcher` asks him to route the request. The parse method returns a `Request` object with the params set accordingly when it matches, and false when it does not.

To get more familiar, we'll add some more assertions to this test, just append them after the last `assertEqual()` and run the tests again. You can see that this route really matches only `/foo` and nothing else:

	$request->url = '/bar';
	$result = $route->parse($request);
	$this->assertFalse($result);

	$request->url = '/foo/edit';
	$result = $route->parse($request);
	$this->assertFalse($result);

Now that we wrote some basic tests, let's move on to a slightly more complex example with a non-static routing template. The route `/users/{:id}` matches with or without an ID given and passes it as a param in the returned request object.

	public function testUserWithIdRoute() {
		$params = array('id' => null, 'action' => 'index', 'controller' => 'users');
		$route = new Route(array(
			'template' => '/users/{:id}',
			'params' => $params
		));
		$request = new Request();

		$request->url = '/users';
		$result = $route->parse($request);
		$this->assertEqual($params, $result->params);
		$this->assertNull($result->params['id']);

		$request->url = '/users/4';
		$result = $route->parse($request);
		$this->assertEqual(4, $result->params['id']);
	}

We can also test for various HTTP parameters like the request type (typically GET, PUT, POST and DELETE). Let's assume we want to match `/cocktails/delete/{:id}` only when the id is present (non-optional), it is numeric and the request method is `delete`. The equivalent router call in our `routes.php` file would be `Router::connect('/cocktails/delete/{:id:\d+}, array('controller' => 'cocktails', 
'action' => 'delete', 'http:method' => 'DELETE'));`.

	public function testCocktailDeleteRoute() {
		$params = array(
			'controller' => 'cocktails',
			'action' => 'delete',
			'http:method' => 'DELETE'
		);
		$route = new Route(array(
			'template' => '/cocktails/delete/{:id:\d+}',
			'params' => $params
		));

		$request = new Request();
		$request->url = '/cocktails/delete/4';
		$result = $route->parse($request);
		$this->assertFalse($result);

		$request = new Request(array('env' => array('REQUEST_METHOD' => 'DELETE')));
		$request->url = '/cocktails/delete';
		$result = $route->parse($request);
		$this->assertFalse($result);

		$request = new Request(array('env' => array('REQUEST_METHOD' => 'DELETE')));
		$request->url = '/cocktails/delete/4';
		$result = $route->parse($request);
		$this->assertEqual(4, $result->params['id']);
	}

As stated in the first lines of this section, you won't need to do this in your application code, but writing this kind of tests has one more benefit: if something goes wrong, it is a great tool to understand the inner workings and also to test scenarios in isolation. If you encounter any bugs in the router, writing a unit test that shows the failure is also highly recommended, so that the core team is able to track down your problem and reproduce it in their environments. Also, writing failing test cases ensures that after a bug is fixed, it never happens again.

A great way to inspect and debug routes is the `export()` method. With this method, you can inspect the full state of an route and run assertions against it. Let's try this in a test:

	public function testExport() {
		$params = array('controller' => 'posts', 'action' => 'index');
		$route = new Route(array(
			'template' => '/read',
			'params' => $params
		));

		$exported = $route->export();
		$this->assertEqual($params, $exported['params']);
		$this->assertEqual('/read', $exported['template']);
		$this->assertFalse($exported['handler']);
	}

Now let's move on to something that we should really use in our applications, integration testing.

### Integration Testing
Now that we've tested each route in isolation, it's time to get a higher-level view of our application. When we test more than one class in isolation, we call that "Integration Testing". Technically, these tests are similar to normal unit tests, so you don't need to learn something new. In a typical framework request, the `Dispatcher` calls the `Router` with a `Request`, and the router matches it against all connected routes and checks if there is a match. 

The following code snippets are suited for a default installation. Create a `RouterTest.php` file in `app/tests/integration/net/http`. Additionally, you may also need to create the appropriate subdirectories in there.

	<?php
	namespace app\tests\integration\net\http;
	use lithium\net\http\Router;

	class RouterTest extends \lithium\test\Unit {

	}

	?>

We can issue a request against the router and check that the output is correct. Let's do that for a subset of our default routes:

	public function testRootRoute() {
		$expected = array('controller' => 'pages', 'action' => 'view');

		$request = new Request();
		$request->url = '/';

		$result = Router::process($request);
		$this->assertEqual($expected, $result->params);
	}

	public function testPagesRoutes() {
		$request = new Request();

		$request->url = '/pages';
		$result = Router::process($request);
		$expected = array(
			'controller' => 'pages',
			'action' => 'view'
		);
		$this->assertEqual($expected, $result->params);

		$request->url = '/pages/1';
		$result = Router::process($request);
		$expected = array(
			'args' => array('1'),
			'controller' => 'pages',
			'action' => 'view'
		);
		$this->assertEqual($expected, $result->params);
	}

	public function testDefaultRoutes() {
		$request = new Request();

		$request->url = '/foo';
		$result = Router::process($request);
		$expected = array(
			'controller' => 'foo',
			'action' => 'index'
		);
		$this->assertEqual($expected, $result->params);

		$request->url = '/foo/bar';
		$result = Router::process($request);
		$expected = array(
			'controller' => 'foo',
			'action' => 'bar'
		);
		$this->assertEqual($expected, $result->params);

		$request->url = '/foo/bar/1';
		$result = Router::process($request);
		$expected = array(
			'controller' => 'foo',
			'action' => 'bar',
			'args' => array('1')
		);
		$this->assertEqual($expected, $result->params);
	}

Most of the code should look familiar to you, the most interesting part is the call of `Router::process()`. This method is also called by the dispatcher (but with the actual Request object). For every call to the Router, we simulate a request by the client browser. This code contains a high amount of duplication, so you may want to compact this and create a helper method in your test class that wraps and simplifies the duplicated code.

Here is another example for a route that we can test:

	use app\models\Post;

	Router::connect('/{:slug:[\w\-]+}', array('Posts::show'), function($request) {

		$conditions = array(
			'conditions' => array(
				'slug' => $request->params['slug']
			)
		);

		if(Post::count($conditions)) {
			return $request;
		} else {
			return false;
		}

	});

When this route is placed right before the default routes, it should correctly match URLs like `/a-cool-article`. The code inside the closure calls the database and checks if the given slug really exists in the database. If it does, it returns the request (and false if not). The router iterates over each route and when false is returned, he tries the next route. So when we return the `Request`-object, we tell the router that our route is the one he has searched for. To ensure that everything works as expected, we can write some integration tests.

	public function testSlugRoute() {
		$request = new Request();

		// a correct slug (stored in the db)
		$request->url = '/a-correct-slug';
		$result = Router::process($request);
		$expected = array(
			'controller' => 'posts',
			'action' => 'show',
			'slug' => 'a-correct-slug'
		);
		$this->assertEqual($expected, $result->params);

		// not a correct slug, renders a default route
		$request->url = '/foobar-not-a-slug-here';
		$result = Router::process($request);
		$expected = array(
			'controller' => 'posts',
			'action' => 'show',
			'slug' => 'foobar-not-a-slug-here'
		);
		$this->assertNotEqual($expected, $result->params);
		$this->assertFalse(isset($result->params['slug']));

		// tests should always come first
		$request->url = '/test';
		$result = Router::process($request);
		$expected = array(
			'controller' => 'lithium\test\Controller',
			'action' => 'index',
		);
		$this->assertEqual($expected, $result->params);
		$this->assertFalse(isset($result->params['slug']));
	}

This code assumes that your database has a reproducible set of posts stored in it. I'd recommend you to load a set of posts with the [li3_fixtures](http://rad-dev.org/li3_fixtures) plugin and then populate the database in the `setUp()` method. If you want to play around with it some more, I'd suggest you to check the current [Environment](http://lithify.me/docs/lithium/core/Environment) and test the different behavior for different environments.

With this tools at hand, you should be able to test everything that has to do with your routes and make sure that they are connected in the correct order and return the response you've intended.

### Reverse Routing
As the router also handles reverse routing (which is used by the HTML helper to generate links), we can test it here too. Because we look at the router and its routes, reverse routing fits well into our router integration tests.

Testing reverse routing is straightforward, so let's start with some examples and work them through afterwards.

	public function testReverseRouting() {
		$expected = '/foo/bar';
		$params = array('controller' => 'foo', 'action' => 'bar');
		$result = Router::match($params);
		$this->assertEqual($expected, $result);

		$expected = '/pages';
		$params = array('controller' => 'pages', 'action' => 'index');
		$result = Router::match($params);
		$this->assertEqual($expected, $result);

		$expected = '/pages/index/1';
		$params = array('controller' => 'pages', 'action' => 'index', 'args' => array('1'));
		$result = Router::match($params);
		$this->assertEqual($expected, $result);
	}

Here, we test the opposite direction. We pass the router a set of arguments and get an URL back. As it's the case with normal
routing, if two routes compete for the same URL, the first one wins. We call `Router::match()` with our request params and get a URL as a string returned. This way, we can test that our HTML helper will return the desired URLs in our application. Of course, you can also add some tests to the HTML Helper (where you can make sure that the echoed HTML code is correct). If the router is not able to translate your params into a URL, it will raise an exception.

### Wrapping up
In part two of our series we've looked at unit testing routes, running integration tests against them and also testing the reverse routing
mechanism. Of course, these tests depend heavily on your application and will surely become more complex as the previous examples. The next higher level would be to do add some acceptance tests with [Selenium](http://seleniumhq.org/), so that you can test routes in combination with links and forms on your page, to make sure that the data flow works as expected between pages. Also, I'm working on a BDD plugin for Lithium that works similar to [Cucumber](http://cukes.info/), so stay tuned.

I really hope that you've enjoyed the second part of this series. Also, thanks again to [@nateabele](http://twitter.com/#!/nateabele) for his helpful comments.