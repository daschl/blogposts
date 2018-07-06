+++
date = "2011-06-08"
draft = false
title = "Securing Lithium Forms"
description = "Recently, Lithium introduced a new feature which lets you secure your forms and protect them from CSRF attacks."
tags = ["php","lithium","security","helper","controller"]
slug = "Securing-Lithium-Forms"
+++

CSRF (Cross-Site-Request-Forgery) attacks work by sending arbitary (form) requests from a victim. Normally, the receiving site (in our case the `Controller` who processes the form data) doesn't know where the data comes from. The CSRF protection in Lithium aims to solve this problem in an elegant and secure way. You can read more about those attacks 
[here](http://shiflett.org/articles/cross-site-request-forgeries). Note that 
you'll need to clone the latest `master` branch of Lithium if you want to 
try it out now.

CSRF protection in Lithium is twofold. First, you need to add a unique token to your forms and then check in your controller if the sent token is correct. As Lithium stores the generated `sessionKey` in your session, make sure that you have session support enabled. If you don't activate sessions, the `check` method fails silently (which we'llsee later on). So uncomment the following line in `app/config/bootstrap.php` (if you haven't already):

	/**
	 * This file contains configuration for session (and/or cookie) storage, and user or web service
	 * authentication.
	 *
	 */
	//require __DIR__ . '/bootstrap/session.php';

Let's look at the view layer first with a very simple form:

	<?= $this->form->create(); ?>
		<?= $this->security->requestToken(); ?>
		<?= $this->form->field('title'); ?>
		<?= $this->form->submit('Submit'); ?>
	<?= $this->form->end(); ?>

Thanks to lazy loading, we don't have to do anything special to 
include our new `Security` helper. The helper provides the 
`requestToken()` method that generates a unique token (a salted hash) and renders it in a hidden field. If you inspect your form, you should see 
something like this (note that i've shortened the value attribute for better 
readability):

	<input type="hidden" value="$2a$1...Ay62/W" name="security[token]">

Now that we've adapted our form, we can work with it in our controller.


	<?php

	namespace app\controllers;

	use lithium\security\validation\RequestToken;

	class TasksController extends \lithium\action\Controller {
		
		public function add() {
			if($this->request->data) {
				if(!RequestToken::check($this->request)) {
					RequestToken::get(array('regenerate' => true));
				} else {
					// work with the request as usual
				}
			}
		}

	}

	?>

There are many ways how to handle security checks, so the code snippet above shows only one of them. Let's tackle the important parts one by one.

	use lithium\security\validation\RequestToken;

You'll need to import the `RequestToken` class into your namespace, as it is the responsible class for dealing with the tokens.

	RequestToken::check($this->request);

The `check` method reads the `sessionKey` from your session and checks if it is identical to the requested one. You can also provide the key directly:

	$key = $this->request->data['security']['token'];
	RequestToken::check($key);

Now you can modify the key manually (set it to `foobar` or so) and then see if the `check` method fails. How you may handle security errors depends heavily on your application. In our example, we regenerate the token with:

	RequestToken::get(array('regenerate' => true));

If you go down this route, you'll also have to tell the user what happend in your view. A more secure route would be to raise an exception (or render a error template) and log what happened. In normal production environments this is clearly a exceptional behavior and therefore should be treated this way. If you need this more often in your controller, you can also move the checks to the `_init()` method.

	class TasksController extends \lithium\action\Controller {
		
		public function _init() {
			parent::_init();
			
			if($this->request->data && !RequestToken::check($this->request)) {
				$host = $this->request->env('HTTP_HOST');
				Logger::error("Possible CSRF attack from host $host");
				$this->redirect('/');
			}
		}
		
		public function add() {
			if($this->request->data) {
					// save your data as usual
			}
		}
	
	}

This should give you a good starting point on how to work with the new 
CSRF protection mechanisms. As of today, there is only one major feature left (namely MongoDB relationships) until Lithium reaches the "big one" so stay tuned for more announcements in the next weeks!
