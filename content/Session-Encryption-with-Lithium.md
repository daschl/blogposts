+++
date = "2012-01-20"
draft = false
title = "Session Encryption with Lithium"
description = "This post shows you how to encrypt your session data transparently and what is going on behind the scenes."
tags = ["php","cookies","session","lithium"]
slug = "Session-Encryption-with-Lithium"
+++

## The Basics
If you check out the `master` branch, you can use the new `Encrypt` strategy to encrypt your session data automatically. This means that you can read and write session data in cleartext and they will be encrypted on the fly before getting stored (in a cookie, for example). You can read my post about "baking cookies like a chef" for  PHPAdvent 2011 [here](http://phpadvent.org/2011/bake-cookies-like-a-chef-by-michael-nitschinger). The article covers both HMAC signatures and encryption, and is a good place to start. This post will go a little bit deeper in how to use and configure the `Encrypt` strategy in Lithium. Note that you need to have the [mcrypt](http://php.net/manual/en/book.mcrypt.php) extension installed and enabled for this to work.

Here's the simplest configuration you can start with:

	Session::config(array('default' => array(
		'adapter' => 'Cookie',
		'strategies' => array('Encrypt' => array('secret' => 'f00bar$l1thium'))
	)));

This will store all your session data in a cookie. If you don't change anything else, your data will be encrypted with [AES](http://en.wikipedia.org/wiki/Advanced_Encryption_Standard) in  [CBC](http://en.wikipedia.org/wiki/Block_cipher_modes_of_operation#Cipher-block_chaining_.28CBC.29) mode with a block size of 256 bits (32 byte). This is a secure and sensible default, so if you don't need anything special just stick with the defaults.

You can now read and write cleartext data and it will be encrypted on the fly for you. For example:

	Session::write('key', 'my secret value');

The result will look like this on the client side:

	appcookie[__encrypted]=c3Rqv%2FFYnuwAevxeseII7lOvyoOTi2foMfB%2FCoO4l4FcutRvBn7qq%2BZrPVWC%2FzEYEiSSFMWv7EhEx4ew99%2FmPQ%3D%3D

You can then read the value again with `Session::read('key');` or delete it with `Session::delete('key');`.

That's it. If you don't want to dig into the internals, you can safely stop reading now and implement it in your own application.

## Further Discussion
Still here? Awesome!

While the `mcrypt` extension does the heavy lifting for us, the `Encrypt` strategy abstracts necessary cruft like vector generation, key strengthening and resource management. While this is okay for most parts of the framework, in my opinion, when it comes to security, you should have a good understanding what is going on under the hood. Code ist not perfect, and it may contain semantical or design flaws that put your whole application (and customers) to risk. Also, security standards shift over time and what is now secure may not be in a few months.

If you don't override the default settings, Lithium encrypts your data with the `MCRYPT_RIJNDAEL_128` cipher in `MCRYPT_MODE_CBC` mode. What
most developers get wrong is the fact that the `128` doesn't represent the strength of the key size, but the block size that is used. Strictly
speaking, `MCRYPT_RIJNDAEL_256` is not part of the AES standard (because AES has a fixed block size of 128 bits and a key size of 128, 192 or 256 bits). The key size is determined by the length of the key that you pass to the `mcrypt` extension.

So to make sure we use the longest key possible, we hash the given secret with `sha256` and then substring the length we need. As `mcrypt` tells us which key lengths are supported for which cipher (through [mcrypt_enc_get_key_size](http://php.net/manual/en/function.mcrypt-enc-get-key-size.php)), it is possible to determine the strongest key length. Note that this doesn't mean that you should use weak keys in the first place just because they get hashed anyway. In practice, you will only create them once so make sure they contain enough entropy.

As always, the Lithium core provides sensible defaults but also lets you outgrow the framework if you need to. You can specify your own cipher or mode like this:

	Session::config(array('default' => array(
		'adapter' => 'Cookie',
		'strategies' => array('Encrypt' => array(
			'cipher' => MCRYPT_RIJNDAEL_256,
			'mode' 	 => MCRYPT_MODE_ECB, // Don't use ECB when you don't have to!
			'secret'	 => 'f00bar$l1thium'
		))
	)));

Here's a list of [ciphers](http://www.php.net/manual/en/mcrypt.ciphers.php) and [modes](http://www.php.net/manual/en/mcrypt.constants.php) you can use. Of course you need to make sure that your custom cipher/mode combination is supported.

If you don't like how Lithium hashes your key or if you want to has it on your own, there's also a way to do this. If the length of your key is equal or longer than the longest key size supported by the algorithm, it gets passed through unchanged. Here's how it works:

	protected function _hashSecret($key) {
		$size = mcrypt_enc_get_key_size(static::$_resource);

		if (strlen($key) >= $size) {
			return $key;
		}

		return substr(hash('sha256', $key, true), 0, $size);
	}

In my opinion, you should always encrypt your data if it gets stored on the client side. Lithium provides a convenient and transparent way to do it, so go ahead and use it in your own applications to make them more secure.