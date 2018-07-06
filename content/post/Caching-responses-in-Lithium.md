+++
date = "2011-05-30"
draft = false
title = "Caching responses in Lithium"
description = "I've hacked together a small example on how to easily cache full responses in Lithium."
tags = ["php","lithium","cache"]
slug = "Caching-responses-in-Lithium"
+++

If you need to cache full `Response` objects in Lithium (which means that your controllers don't even get called when there's a cache hit), you can place this in your `app/config/bootstrap/cache.php` file (note that this is certainly not "production ready", but it should give you a starting point):

	/**
	 * Cache full Responses
	 */
	Dispatcher::applyFilter('run', function($self, $params, $chain) {
		$key = md5(LITHIUM_APP_PATH) . '.app.cache.'.md5($params['request']->url);
		if($cache = Cache::read('default', $key)) {
			return $cache;
		}

		$result = $chain->next($self, $params, $chain);

		Cache::write('default', $key, $result, '+1 day');
		return $result;
	});

It filters the `Dispatcher` and first checks if the `Response` object is already stored in the cache. The cache key is based on the requested URL, so if your `/` URL points to ``/posts/index`, both requests will be cached seperately (you could modify the code snipped so that it talks to the `Router` and checks for the corresponding controller/action object and cache on this key).

Your filter chain returns the `Response` object, so we can directly cache it. The serialization strategy takes care of correctly storing and retrieving our cached object.

If you have alternative approaches to tackle this, please let me know!  
