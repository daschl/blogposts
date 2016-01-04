+++
date = "2011-06-03"
draft = false
title = "Custom Finders with Lithium"
description = "In this post we'll take a look at custom finders and how they can help you DRYing up your code."
tags = ["php","lithium","model","finders","mysql"]
slug = "Custom-Finders-with-Lithium"
+++

Finders assist you with often-used database queries so you don't have to write them over and over again. Out of the box, Lithium provides you with a bunch of them: `all`, `first`, `count` `list` and "magic finders like" 
`findById` or `findFirstById`. How these are constructed in the core is not 
relevant for now, but Lithium provides you with a mechanism to write your own finders easily.

To understand how we can implement our own finder, let's take a look at the built-in `first` one. If you want to dig deeper, look at the 
[_findFilters()](http://lithify.me/docs/lithium/data/Model::_findFilters%28%29) method. The `_findFilters()` method constructs the default finders and returns them to the `Model::config()` method so that they are loaded when your model is initialized. 

	// ... some php code ...
	'first' => function($self, $params, $chain) {
		$params['options']['limit'] = 1;
		$data = $chain->next($self, $params, $chain);
		$data = is_object($data) ? $data->rewind() : $data;
		return $data ?: null;
	}
	// ... more finders ...

Your custom finders will also take the same params, so they let's investigate them:

- `$self` contains the current model as a string
- `$params` contains the find options that you already know like `order` 
or `conditions`.
- `$chain` contains filter chain.

As you may have recognized, this looks like our typical 
[Filters](http://rad-dev.org/drafts/source/en/02_lithium_basics/02_filters.wiki) pattern in Lithium! In fact, all finders run through the filter system and this is good news for us: we can also adapt the default ones if we need to. 

Our code example first sets a `limit` option and then lets the chain move on. At the end of the chain, it sets the returned data to the beginning if its an object or just returns the array. Now it's time to write our own finder. To keep things simple at first, let's say we want to find the first five datasets in our model. Just add it to your controller for now, we'll see a bit later how to move it to your model.

	Tasks::finder('firstFive', function($self, $params, $chain) {
		$params['options']['limit'] = 5;
		$data = $chain->next($self, $params, $chain);
		return $data ?: null;		
	});

In our example we have a `Tasks` model who just has an `id` and a task `name` stored. Note that we use MySQL here so that noone can say I'm always using MongoDB ;). We can now run our finder with one of the following methods:

	$tasks = Tasks::find('firstFive')
	$tasks = Tasks::firstFive();
	
If you want to check the results immediately, you can use the handy `to('array')` method on the result set:

	var_dump(Tasks::firstFive()->to('array'));
	
	// Should print something like
	array
		1 => 
			array
				'id' => string '1' (length=1)
				'title' => string 'first task' (length=10)
		2 => 
			array
				'id' => string '2' (length=1)
				'title' => string 'second task' (length=11)
		3 => 
			array
				'id' => string '3' (length=1)
				'title' => string 'third task' (length=10)
		4 => 
			array
				'id' => string '4' (length=1)
				'title' => string 'fourth task' (length=11)
		5 => 
			array
				'id' => string '5' (length=1)
				'title' => string 'fifth task' (length=10)

If you comment out the limit line in our filter, you can see that you get all results back from the database (just make sure you have more than five rows stored so you can see the difference).

Recently, a new functionality was added to make custom finders a bit more flexible. You can now just hand over an array of params and you don't need to provide a full closure. This comes in handy when you just want to add a `limit` or a fixed set of `conditions`. Note that this is currently only available in the `x-relationships` branch. Once the whole relationship-related code gets merged into master, this feature will also be available.

	Tasks::finder('myAwesomeFinder', array(
		'fields' => array('id', 'name'),
		'conditions' => array('id' => 2)
	));
	
	$tasks = Tasks::myAwesomeFinder();
	
You can also add the custom finder in your model, so you don't have to deal with it in the controller. According to the MVC pattern, I'd recomend you to place it there. You can override the `__init()` method so that your finder will be loaded at runtime.

	public static function __init() {
		parent::__init();
		static::finder('firstFive', function($self, $params, $chain) {
			$params['options']['limit'] = 5;
			$data = $chain->next($self, $params, $chain);
			return $data ?: null;
		});
	}

I think this is enough for a short introduction into custom finders, if you want to find out more you should check out the [Model::finder](http://lithify.me/docs/lithium/data/Model::finder%28%29) 
method and poke around in the [Model::find](http://lithify.me/docs/lithium/data/Model::find%28%29) source. Once the model part of the [drafts](http://rad-dev.org/drafts/source) is finished, you'll finde more detailed documentation on this topic. 

With this tool in mind, interesting possibilities start to pop up. You can now easily add logging during development to your finders, add caching mechanisms and so on. Please cast your vote in the comments if you want to read more on this topic. So now go ahead and [DRY](http://de.wikipedia.org/wiki/Don%E2%80%99t_repeat_yourself) up your controllers and models!