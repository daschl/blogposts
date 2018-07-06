+++
date = "2010-11-19"
draft = false
title = "Lithium Routes And MongoDB"
description = "In my first post I'll show you how to use li3-routes in combination with MongoDB ObjectIDs."
tags = ["php","lithium","mongodb","routing"]
slug = "Lithium-Routes-And-Mongo-DB"
+++

Lithium comes with a powerful and flexible routing system. However, in its default configuration, you may encounter some problems with MongoDB-ObjectIDs and reverse routing. ObjectIDs are usually the "primary keys" of your Document in MongoDB, are 12 bytes long and consist of numbers and characters from 'a' to 'f'. A typical ObjectID would look similar to `4ce2d9f99436485e05000000`.

If you take a closer look at the default routes that ship with Lithium, you'll maybe notice that the `:id` part only matches numbers (and not characters).

    Router::connect('/{:controller}/{:action}/{:id:[0-9]+}.{:type}', array('id' => null));
    Router::connect('/{:controller}/{:action}/{:id:[0-9]+}');
    Router::connect('/{:controller}/{:action}/{:args}');

With this setup a call to `$this->html->link()` with an MongoDB-ObjectID set as `id` won't work because the router is not able to find a matching route (this is called reverse routing). The solution to this "problem" is, that we need to modify the `:id`-part of the `Router::connect`-calls. Replace your default routes with the following code:

    Router::connect('/{:controller}/{:action}/{:id:[0-9a-f]{24}}.{:type}', array('id' => null));
    Router::connect('/{:controller}/{:action}/{:id:[0-9a-f]{24}}');
    Router::connect('/{:controller}/{:action}/{:args}');

Now everything should work as expected with li3 and MongoDB ObjectIDs. A final note for the interested reader: if you leave out the `{24}`-quantifier, then a route like `/posts/add` won't work because `add` would also be a valid `id` to the router.