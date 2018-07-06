+++
date = "2012-02-24"
draft = false
title = "RFC: li3_fixtures Rewrite"
description = "Read what I plan for the new li3_fixtures plugin and share your thoughts!"
tags = ["php","li3_fixtures","plugin","lithium"]
slug = "RFC-li3_fixtures-Rewrite"
+++

The [li3_fixtures](https://github.com/daschl/li3_fixtures) plugin was my first Lithium plugin ever, and while it works okay, I feel there is a lot I can do to make it better and more flexible. In this post I want to share my ideas for a new fixture plugin and also want to gather feedback from the community to make it even more awesome.

As far as I can see, there are three big use cases for fixtures:

## Unit-Testing Models
This seems to be the most common use case. To test models effectively, you need a bunch of demo data on which your tests rely on. This data includes both valid and invalid data to test if validations work as expected and how the application can handle large batches of data. The database itself is not mocked, but fixture data is used to populate it before or during the actual tests.

This can be done without fixtures too, but you need to reinvent the wheel and store your demo data either in external files or directly in your code. When you store it in external files, you need to mess around with file loading and parsing. If you store it directly in your code it gets bloated and makes your code unreadable (and your data is always bound to the test class and can't be reused). To make it as easy as possible, the `li3_fixtures` plugin needs to provide the following:

- centralized storage of fixture data.
- transparent support for various data types (JSON, XML and YAML).
- easy loading of fixture datasets into the database.

## Mocking Models
When you test your controllers, you often need to work with data from your models. There are two different scenarios here: you can either write integration tests and populate your real database with data, or mock your models away and work with static data instead of touching the database (for unit-testing controller actions). This allows you to work independently on both layers and improves the performance of your tests.

Lithium already uses Mocks extensively, so the plugin needs tight integration into them. It should be easy to load fixture data that acts like "real" data (like iterating over Document/RecordSets).

## Mocking Web-Services and APIs
The third use case is not really different from the two mentioned above, but works more with the `service` than the the `data` layer. As web services are often not inside the boundaries of the developer, the plugin should make it easy to provide demo data for mocks while testing web services.

## Additional Functionality
In terms of rapid application development, it would be great to generate fixture data automatically based on patterns or generation rules. There are great libraries like [Faker](https://github.com/fzaninotto/Faker) out there, and it would be great to have a feature like this integrated (accessible from the command line). Also, fixture file generation should happen automatically when new models are created from the command line.

## Implementation & Documentation
I want to make extensive use of adapters and strategies, as they make a lot of sense in this context. I think it would be a good idea to use adapters for all input file types (JSON, XML and YAML) and strategies for their representation in the test classes (DocumentSets, RecordSets, Collections or plain arrays). This also makes it trivial to implement custom file types and representations.

For documentation, I plan to use li3_docs for both the API docs and the actual manual. A current version of the manual will be found as a subdomain on my blog here (of course you'll also have it in your app when you use li3_docs).

Finally, the plugin will be usable through composer and uses it too to manage external dependencies (I plan to use the Symfony parser for the YAML format).

## Next Steps
Now it's your turn! Comment below and tell me wheter you like the ideas presented here or not. I'm open to every suggestion!
