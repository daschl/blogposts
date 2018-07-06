+++
date = "2011-09-16"
draft = false
title = "Quick Tip: Lithium Redirect"
description = "Learn how to correctly redirect URLs based on the Request information."
tags = ["php","redirect","router","lithium"]
slug = "Quick-Tip-Lithium-Redirect"
+++

While migrating [lithium_bin](https://github.com/UnionOfRAD/lithium_bin) as part of research over to MongoDB (from CouchDB), I found the following snippet in the `routes.php` file:

	Router::connect('/', array(), function($request) {
		$location = array('controller' => 'pastes', 'action' => 'add');
		return new Response(compact('location'));
	});

This means that when the user enters the application via the root url (`/`), he instantly gets redirected to `/pastes/add` (or a different URL if you have custom routes configured).

This may seem ok at first, but there's a problem. It doesn't take URLs into account that don't live directly under the document root. So if your application lives under `http://localhost/pastium/`, it will redirect you to `http://localhost/pastes/add/` which is not really what you want. Instead, do it like this:

	Router::connect('/', array(), function($request) {
		$location = Router::match('Pastes::add', $request);
		return new Response(compact('location'));
	});

In this snippet, the `Router` takes the current `Request` into account and returns the correct location (based on the reverse routing information). Now it should redirect you correctly to `http://localhost/pastium/pastes/add/`.

By the way, this is extracted from the `Controller::redirect` method, which is implemented in a similar way:

	public function redirect($url, array $options = array()) {
		//...
		$this->_filter(__METHOD__, $params, function($self, $params) use ($router) {
			$options = $params['options'];
			$location = $options['location'] ?: $router::match($params['url'], $self->request);
			$self->render(compact('location') + $options);
		});
		//...
	}

If you have any ideas on how to improve this further, feel free to comment below!