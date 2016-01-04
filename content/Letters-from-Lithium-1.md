+++
date = "2011-08-02"
draft = false
title = "Letters from Lithium #1"
description = "Letters from Lithium keeps you up to date with Lithium and its community. Issue #1 2011-08-02"
tags = ["php","lfl","lithium"]
slug = "Letters-from-Lithium-1"
+++

About
-----
"Letters from Lithium" will be published on a regular basis to give you an update on what happens in the Lithium community. If you have interesting topics/plugins/stuff you want to read about, feel free to contact me through twitter or email.

Core News
---------
The [Roadmap](https://github.com/UnionOfRAD/lithium/wiki/Roadmap) to 1.0 is going pretty well so far, and relationship support for relational databases has been already merged into master. For the impatient, there is also some documentation for it underway which can be found [here](https://github.com/UnionOfRAD/manual/blob/master/en/working-with-data/relationships.wiki). As the docs are not finished yet, you'll have to clone the repository locally to view them in their full glory (don't forget to clone `li3_docs` too). The core team is also working on relationship support for MongoDB, which will also ship before 1.0. For those who come from CakePHP: there won't be a `hasAndBelongsToMany` relationship, because we'll add support for nested relationships which can be used to achieve the same result in a better and cleaner way.

We're very excited to see more and more people contributing code to the core. [Chris Beswick](https://github.com/doobry) did an awesome job recently and refactored a significant part of the routing system ([commits](https://github.com/UnionOfRAD/lithium/commits/master?author=doobry)) which makes compiling routes significantly faster. Also there's a huge [pull request](https://github.com/UnionOfRAD/lithium/pull/45) open that brings [PostgreSQL](http://www.postgresql.org/) integration to the Lithium core.

Lithium wants to be a first-class citizen on Windows, so there's a [branch](https://github.com/UnionOfRAD/lithium/tree/windows) open that tries to track, find and fix all bugs and test-cases on Windows. If you're developing on Windows and have some spare time, contact one of the core members on IRC and we'll get you started. We aim to make all test cases run on Windows before the 1.0 release.

Plugins
-------
I've recently [compiled a list of plugins](http://nitschinger.at/Lithium-plugin-roundup) and readers also posted their additions in the comments below. There are plans to bring the [Lithium Lab](http://lab.lithify.me) up to date and integrate it even better with the Lithium framework so in the future you should be able to install and share plugins right from there. I'll cover this in greater detail in one of the next letters.

[Alexander Morland](https://github.com/alkemann) is developing a [plugin](https://github.com/alkemann/li3_zmq) for [ZeroMQ](http://www.zeromq.org/) and is always searching for feedback and contributions. If you need message passing in your application (like queues or pub/sub), then you should definitely check it out.

If you don't know which plugins are available, try [this](https://github.com/search?langOverride=&language=PHP&q=li3_&repo=&start_value=1&type=Repositories&x=32&y=21) search on Github!

Blogs & Media
-------------
For those who don't know yet, [Raymond Julin](http://raymondjulin.com/) did compile a list of useful resources back in January which can be found [here](http://blog.raymondjulin.com/2011/01/06/list-of-useful-lithium-li3-resources/). In addition to that, [Tom Maiaroto](http://www.shift8creative.com/about) wrote a [blog post](http://www.shift8creative.com/blog/getting-started-with-minerva) that helps you to get up and running with [Minerva](https://github.com/tmaiaroto/minerva), his CMS based on Lithium.

Lithium now has [Vimeo Channel](http://vimeo.com/channels/li3) which also contains the [Introducing Lithium](http://vimeo.com/26183322) talk by Nate. You should definitely check that out if you're new to Lithium and need a one-hour introduction.

Universe
--------
As a side note, the [UnionOfRAD](http://www.union-of-rad.com/) is looking for client work (preferrably in the NY area), so if you need talented developers that know Lithium and its surroundings from the inside out please contact [Nate Abele](http://www.nateabele.com) on [twitter](http://www.twitter.com/nateabele) or IRC.