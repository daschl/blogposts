+++
date = "2010-11-20"
draft = false
title = "Add an RSS-Feed to your blog with Lithium"
description = "A common feature of every blog is a RSS-Feed. In this post I'll show you how to do this easily with Lithium and its View-Layer."
tags = ["php","lithium","rss","view"]
slug = "Add-an-RSS-Feed-to-your-blog-with-Lithium"
+++

Adding an RSS-Feed to your blog with li3 is really straightforward, because all the infrastructure you need is already at your fingertips. I assume that you already have some posts stored in your database, you serve them to your users via `/posts` and the new RSS-File should be accessible via `/posts.rss`.

First, we need to tell Lithium that we want to add another Media-Type to our application. Add the following to your `app/config/bootstrap/media.php`-File (and don't forget to uncomment it in `app/config/bootstrap.php`). You can also comment the other code out in this file if you don't need it.

    use lithium\net\http\Media;
    Media::type('rss', 'application/rss+xml');

Next, let's add a link to our `default.html.php`-Layout so that our feed shows up in the browser.

    <?= $this->html->link('RSS-Feed', '/posts.rss', array('type' => 'rss')); ?>

Before we start to create our RSS-Views, check that your `PostsController::index()`-Method looks something like this and returns one or more posts to the View.

	public function index() {
		$posts = Post::all(array('order' => array('created' => 'DESC')));
		return compact('posts');
	}

Alright, first create a new `default.rss.php`-Layout (in `app/views/layouts`):

    <?php echo '<?xml version="1.0" encoding="UTF-8" ?>'; ?>
		<rss version="2.0">
			<channel>
				<title>My cool Blog</title>
				<description>A short description about my blog.</description>
				<link>http://exampleblog.com/</link>
				<lastBuildDate><?= date('D, d M Y g:i:s O'); ?></lastBuildDate>
				<pubDate><?= date('D, d M Y g:i:s O'); ?></pubDate>
				<?= $this->content(); ?>
			</channel>
		</rss>

The previous code is just basic RSS-Markup. Feel free to add a custom title, description, link and so on. The `lastBuildDate` and `pubDate` are generated dynamically. `<?= $this->content(); ?>` will render our `posts`-View which we'll add now. Insert the following code in `app/views/posts/index.rss.php`:

		<?php foreach($posts As $post): ?>
		<item>
			<title><?= $post->title; ?></title>
			<description><?= $post->teaser; ?></description>
			<link><?= $this->html->link($post->title, array('Posts::show', 'slug' => $post->slug)); ?></link>
			<guid isPermaLink="true">http://exampleblog.com/<?= $post->slug ?></guid>
			<pubDate><?= date('D, d M Y g:i:s O', $post->created->sec); ?></pubDate>
		</item>
		<?php endforeach; ?>

You may need to modify the attributes according to your database schema. If you need help on this, feel free to contact me or ask in #li3 on FreeNode.

That's it! You've added an RSS-Feed to your blog. You can now click the RSS-Icon in your browser or render it directly with `http://exampleblog.com/posts.rss`. BTW: that's exactly how I did it for this blog, so I hope that it should work for you too!