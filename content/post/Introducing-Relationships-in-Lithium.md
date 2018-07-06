+++
date = "2012-03-03"
draft = false
title = "Introducing Relationships in Lithium"
description = "This post introduces you to model relationships in Lithium."
tags = ["php","relationships","lithium","models"]
slug = "Introducing-Relationships-in-Lithium"
+++

# Introduction
The model relationship support in Lithium is one of the hottest topics on IRC lately, so I thought it would be a good idea to blog about it.

Currently, Lithium supports 1:1 and 1:n relationships for relational databases. There is no m:n support out of the box (like CakePHP's `$hasAndBelongsToMany`). This also means that MongoDB relationships are not implemented for now. If you look at the [Roadmap](https://github.com/UnionOfRAD/lithium/wiki/Roadmap), you can see that this is on the "pre 1.0" list and will be addressed before the 1.0 version is released.

This post gives you a little background on relationship types and their database representations before we implement a simple example in PHP.

# Basics First
If you know how the three "big" relationship types 1:1, 1:n and m:n are implemented on the database side, feel free to jump straight to the practical example. This is not a exhaustive guide to the relational database model and should only set the stage for the practical example later. Note that I didn't add any foreign key constraints on the database level to make it easier to follow along.

If you work with a relational data model, and have a (more or less) [normalized](http://en.wikipedia.org/wiki/Database_normalization) schema, you need to find a way to model relationships between relations (speak "entities" or "tables"). In pratice, these relationships fall into three categories:

## 1:1 Relationships
Exactly one entity belongs to exactly one other entity. For example, think of a `people` relation and you want to model 
the relationship between married people. One husband can only belong to one wife and vice versa (depending from where you are reading this post this might be different ;)). In a physical database schema, it may look like this.

	mysql> describe people;
	+-----------+--------------+------+-----+---------+----------------+
	| Field     | Type         | Null | Key | Default | Extra          |
	+-----------+--------------+------+-----+---------+----------------+
	| id        | int(11)      | NO   | PRI | NULL    | auto_increment |
	| name      | varchar(255) | YES  |     | NULL    |                |
	| spouse_id | int(11)      | YES  |     | NULL    |                |
	+-----------+--------------+------+-----+---------+----------------+
	3 rows in set (0.01 sec)

The `spouse_id` references a different person (note that it is not important if you reference an entity in the same or a different table. Here's some example data:

	mysql> select * from people;
	+----+--------+-----------+
	| id | name   | spouse_id |
	+----+--------+-----------+
	|  1 | Bob    |         2 |
	|  2 | Alice  |         1 |
	+----+--------+-----------+
	4 rows in set (0.00 sec)

`Bob` and `Alice` are married and their relationship is identified over the foreign key `spouse id`. In Lithium, this type of relation is called `hasOne`.

## 1:n Relationships
Exactly one entity belongs to zero or more other entites. This is easy: a user can write zero or more blog posts. Let's model this accordingly:

	mysql> describe posts;
	+---------+--------------+------+-----+---------+----------------+
	| Field   | Type         | Null | Key | Default | Extra          |
	+---------+--------------+------+-----+---------+----------------+
	| id      | int(11)      | NO   | PRI | NULL    | auto_increment |
	| title   | varchar(255) | NO   |     | NULL    |                |
	| user_id | int(11)      | NO   |     | NULL    |                |
	+---------+--------------+------+-----+---------+----------------+
	3 rows in set (0.01 sec)

	mysql> describe users;
	+-------+--------------+------+-----+---------+----------------+
	| Field | Type         | Null | Key | Default | Extra          |
	+-------+--------------+------+-----+---------+----------------+
	| id    | int(11)      | NO   | PRI | NULL    | auto_increment |
	| name  | varchar(255) | NO   |     | NULL    |                |
	+-------+--------------+------+-----+---------+----------------+
	2 rows in set (0.01 sec)

Here's some example data to work with:

	mysql> select * from posts;
	+----+--------------+---------+
	| id | title        | user_id |
	+----+--------------+---------+
	|  1 | First post.  |       1 |
	|  2 | Second post. |       1 |
	+----+--------------+---------+
	2 rows in set (0.00 sec)

	mysql> select * from users;
	+----+-----------+
	| id | name      |
	+----+-----------+
	|  1 | Bob       |
	|  2 | Alice     |
	+----+-----------+
	1 row in set (0.00 sec)

It's easy to see that Bob wrote two posts while Alice didn't write a single one. We can look at these relations from two different angles (and can also answer two different questions): "What posts did Bob write?" and "Who is the author of the first post?". To answer both questions 
 orrectly, Lithium uses two different relation types for the models: `hasMany` and `belongsTo`. One user `hasMany` blog posts, but one post `belongsTo` only one user. The wording also implies where the foreign key is placed: in a `hasMany` relationship, the foreign key is expected on the other model, while `belongsTo` expects it in the current model. We'll see shortly how this works out in practice.

## m:n Relationships
Lastly, zero or more entities can belong to zero or more other entities. This is a bit tricky, because we need a third table to model this relationship efficiently. For example, a user can belong to zero or more groups and groups can have zero or more members.

	mysql> describe users;
	+-------+--------------+------+-----+---------+----------------+
	| Field | Type         | Null | Key | Default | Extra          |
	+-------+--------------+------+-----+---------+----------------+
	| id    | int(11)      | NO   | PRI | NULL    | auto_increment |
	| name  | varchar(255) | NO   |     | NULL    |                |
	+-------+--------------+------+-----+---------+----------------+
	2 rows in set (0.02 sec)

	mysql> describe groups;
	+-------+--------------+------+-----+---------+----------------+
	| Field | Type         | Null | Key | Default | Extra          |
	+-------+--------------+------+-----+---------+----------------+
	| id    | int(11)      | NO   | PRI | NULL    | auto_increment |
	| name  | varchar(255) | YES  |     | NULL    |                |
	+-------+--------------+------+-----+---------+----------------+
	2 rows in set (0.02 sec)

	mysql> describe users_groups;
	+----------+---------+------+-----+---------+-------+
	| Field    | Type    | Null | Key | Default | Extra |
	+----------+---------+------+-----+---------+-------+
	| group_id | int(11) | NO   |     | NULL    |       |
	| user_id  | int(11) | NO   |     | NULL    |       |
	+----------+---------+------+-----+---------+-------+
	2 rows in set (0.01 sec)

You can see that we need a third table to "glue" the other two together. Here's some example data:
	
	mysql> select * from users;
	+----+-----------+
	| id | name      |
	+----+-----------+
	|  1 | Alice     |
	|  2 | John      |
	|  3 | Tom       |
	+----+-----------+
	3 rows in set (0.00 sec)

	mysql> select * from groups;
	+----+----------------+
	| id | name           |
	+----+----------------+
	|  1 | Administrators |
	|  2 | Support        |
	+----+----------------+
	2 rows in set (0.00 sec)

	mysql> select * from users_groups;
	+----------+---------+
	| group_id | user_id |
	+----------+---------+
	|        1 |       1 |
	|        1 |       2 |
	|        2 |       2 |
	+----------+---------+
	3 rows in set (0.00 sec)

`John` has access to both groups, while the `Administrator` group has two members. Lithium doesn't support this kind of relationship for now, but in CakePHP it is called `hasAndBelongsToMany`.

# The Blog example
Now that we've got the basics settled, let's get our hands on some code and implement the "Hello World 2.0" example - 
a blog. Along the way we can reuse some of the tables above.

Let's assume we want to store blog posts and their authors. For simplicity, every author can belong to only one group and a group can only have one member. This way, we can practice 1:n and 1:1 relationships. Let's create the appropriate data model using the MySQL DDL:

	CREATE TABLE groups (
		id integer auto_increment primary key,
		name varchar(255) not null
	);
		
	CREATE TABLE authors (
		id integer auto_increment primary key,
		firstname varchar(255) not null,
		lastname varchar(255) not null,
		group_id integer
	);

	CREATE TABLE posts (
		id integer auto_increment primary key,
		title varchar(255) not null,
		body text,
		author_id integer
	);
	
Let's insert some demo data to work with:

	insert into groups (name) values ('Admins');
	insert into groups (name) values ('Editors');

	insert into authors (firstname, lastname, group_id) values ('Johnnie', 'Walker', 1);
	insert into authors (firstname, lastname, group_id) values ('John', 'Jameson', 2);
	insert into authors (firstname, lastname, group_id) values ('Jack', 'Daniels', 2);

	insert into posts (author_id, title, body) values (1, 'On whisky nosing', '...');
	insert into posts (author_id, title, body) values (2, 'Irish on the ROCKS', '...');
	insert into posts (author_id, title, body) values (2, 'Without fear.', '...');
	insert into posts (author_id, title, body) values (3, 'On drinking Bourbon', '...');

Next, grab a fresh copy of Lithium and create a connection in `config/bootstrap/connections.php`:

	Connections::add('default', array(
		'type' => 'database',
		'adapter' => 'MySql',
		'host' => 'localhost',
		'login' => '<USER>',
		'password' => '<PASSWORD>',
		'database' => '<DBNAME>',
		'encoding' => 'UTF-8'
	));

Now, let's create three bare-bone models in `app\models` to start with:

	<?php
	// Filename: /app/models/Posts.php
	namespace app\models;

	class Posts extends \lithium\data\Model {

	}
	?>

	<?php
	// Filename: /app/models/Authors.php
	namespace app\models;

	class Authors extends \lithium\data\Model {

	}
	?>

	<?php
	// Filename: /app/models/Groups.php
	namespace app\models;

	class Groups extends \lithium\data\Model {

	}
	?>

To produce meaningful output in the browser, we'd better add a controller and a view template.

	<?php
	// Filename: /app/controllers/PostsController.php
	namespace app\controllers;

	use app\models\Posts;
	use app\models\Authors;
	use app\models\Groups;

	class PostsController extends \lithium\action\Controller {

		public function index() {
			
		}
		
	}
	?>


	<!-- Filename: /app/views/posts/index.html.php -->
	<h2>All Posts.</h2>
	
	More stuff will be here soon.

Now that we have everything in place, we can define our relationships in Lithium. Place the following snippets in their appropriate models:

	// Posts:
	public $belongsTo = array('Authors');
	
	// Authors:
	public $hasMany = array('Posts');

	// Groups:
	public $hasOne = array('Authors');

As our database tables rely on conventional column names, we don't have to provide more configuration params to Lithium. Now, let's read all posts, their corresponding authors and print them in the template.

	// PostsController:
	public function index() {
		$posts = Posts::all(array('with' => 'Authors'));
		return compact('posts');
	}


	// index.html.php:
	<ul>
		<?php foreach($posts as $post): ?>
			<li>
				<strong><?= $post->title; ?></strong>, by 
				<?= $post->author->firstname; ?> <?= $post->author->lastname; ?>
			</li>
		<?php endforeach; ?>
	</ul>

The `with` param controls what relationships you want to fetch. In this case, as a post belongs to one author, Lithium knows that only one `Record` is returned and you can use the `author` directly. If you don't provide the appropriate class there, Lithium won't call the associated model and an error will be raised (because `$post->author` isn't set, PHP complains that you want to access `firstname` from a non-object). This is a common mistake, so keep it in mind!

Lithium is very flexible on the data layer, and you can write instance methods on your model and access them through the relation. Let's say we want to provide the full name at the model layer, so that we don't have to read both the first- and the lastname in the template.

	// Authors:
	public function fullName($entity) {
		return $entity->firstname." ".$entity->lastname;
	}

	// index.html.php:
	<strong><?= $post->title; ?></strong>, by <?= $post->author->fullName(); ?>

We can also read all posts that belong to a particular author:

	$posts = Authors::first(array(
		'conditions' => array('posts.id' => $author_id), 
		'with' => 'Posts'
	))->posts;

To find data through our 1:1 relation, we can use the same mechanisms:

	// Find the author name for a group:
	$name = Groups::first(array(
		'conditions' => array('groups.id' => $group_id),
		'with' => array('Authors')
	))->author->fullName();

Note that if we want to find the name of the associated group for an author, we would need to add a foreign key to the `groups` table (there is no real difference between `hasOne` and `hasMany`, just the fact that `hasOne` returns exactly one entity instead of zero or more and therefore no `Collection` is needed).

# Wrapping up
This post has covered the basics of model relationships and how to model the 1:1 and 1:n types in Lithium. We now set the stage for more advanced topics, including: form integration, faking the m:n relationship for relational databases, faking relationships for MongoDB and advanced relationship handling through the `Relationship` class. If you have any other interesting topics that you want to read about soon, share your ideas!