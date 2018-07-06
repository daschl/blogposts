+++
date = "2012-05-10"
draft = false
title = "Writing a simple lexer in PHP"
description = "This post shows you how to write a simple regex lexer in PHP."
tags = ["php","regex","lexer","parser"]
slug = "Writing-a-simple-lexer-in-PHP"
+++

# Introduction
A lot of developers avoid writing parsers because they think it's pretty hard to do so. Writing an efficient parser for a general purpose language (like PHP, Ruby, Java,...) is hard, but fortunately, most of the time we don't need that much complexity. Typically we just want to parse input coming from config files or from a specific problem domain (expressed through DSLs). DSLs (Domain Specific Languages) are pretty cool, because they allow you to express logic and flow in a very specific and convenient way for a limited set of tasks.

For example, if you want to map a Route in Lithium to a controller/action, it looks like this:

	Router::connect('/login', array('Sessions::add'));

Pretty short, but we can do better. Consider the following DSL syntax I just made up:

	map /login -> Sessions::add

By using a notation like this, it is easer to grasp what's going on, because we can avoid all the PHP specifics like its array or method syntax. Another benefit is that someone who can write a DSL file doesn't need to know the underlying language which implements it. If a Java-guy likes our language, he can write a parser for it as well and the DSL doesn't need to change at all. In our example it doesn't make sense, but the [Gherkin](https://github.com/cucumber/gherkin) DSL for behaviour driven development has been ported to various languages. The original implementation was done in Ruby for the [Cucumber](http://cukes.info/) project and the [Behat](http://behat.org/) project ported it over to PHP.

Okay, back to our example. This is cool, but how can we transform the latter line, so that it executes our original PHP command? We can either explode it by whitespace and then mess around finding the correct keywords. Or, we can write write an interpreter. A compiler (or interpreter) normally consists mainly of two things: a lexer and a parser. The lexer transforms the raw source into a stream of tokens on which the parser operates on. A parser is normally described by a formal notation like [EBNF](http://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_Form) or something along that lines and knows what to do when a specific stream of tokens is found.

Let's take our example from above. The `map /login -> Sessions::add` may be translated into the following token stream by the lexer (for the sake of simplicity we just display them as tuples):

	<T_MAP, "map">
	<T_URL, "/login">
	<T_BLOCKSTART, "->">
	<T_IDENTIFIER, "Sessions::add">

The parser can parse the following notation (note that this is not EBNF, I've added regex in `[]` to keep it simple):

	<whitespace> 	 := [\s]
	<map> 		 	 := "map"
	<url> 		 	 := [a-z/]
	<blockstart> 	 := "->"
	<identifier> 	 := [a-zA-Z0-9:]
	<mapBlock> 	 	 := <map> <whitespace>* <url>+ <whitespace>* <blockstart> <whitespace>* <identifier>+

Every rule identified by a `:=`is called a production rule. I leave the parsing part for a later post, because that would be too much for a single posting. Nevertheless, you can see that if the parser finds a `mapBlock`(which consists of smaller production rules), it knows the `identifier` and can transform it into the appropriate `Router::connect()` call.

Note that is up to you if you ignore whitespace in your lexer. This depends on design decisions like if you want to use significant whitespace or not. If you parse it as a separate token, make sure your parser is intelligent enough to skip them later.

Enough theory, let's get started with the lexer.

# Implementation
The basic idea is that we match a list of regexes against the current line. If one of them matches, we store that token and advance our offset to the first character after the match. If no token is found and we are not yet at the end of the line, raise an exception (because something invalid is in front of our offset).

Let's stick with the example already mentioned above. We want to create tokens for an input file like `map /login -> Sessions::add` or `root -> Pages::home`.

The `root` command maps the `/`URL (like `Router::connect('/', array('Pages::home'));` in Lithium). The first thing we need is an array of [terminal symbols](http://en.wikipedia.org/wiki/Terminal_and_nonterminal_symbols#Terminal_symbols) that map to token identifiers.

	<?php
	class Lexer {
		protected static $_terminals = array(
			"/^(root)/" => "T_ROOT",
			"/^(map)/" => "T_MAP",
			"/^(\s+)/" => "T_WHITESPACE",
			"/^(\/[A-Za-z0-9\/:]+[^\s])/" => "T_URL",
			"/^(->)/" => "T_BLOCKSTART",
			"/^(::)/" => "T_DOUBLESEPARATOR",
			"/^(\w+)/" => "T_IDENTIFIER",
		);
	}
	?>

Now, let's implement a `run` method that accepts an array of source lines and returns an array of tokens. It calls the helper `_match` method that performs the actual matching and raises an exception if no token was found.

	public static function run($source) {
		$tokens = array();
		
		foreach($source as $number => $line) {            
			$offset = 0;
			while($offset < strlen($line)) {
				$result = static::_match($line, $number, $offset);
				if($result === false) {
					throw new Exception("Unable to parse line " . ($line+1) . ".");
				}
				$tokens[] = $result;
				$offset += strlen($result['match']);
			}
		}
		
		return $tokens;
	}

Note that how we advance the offset further every iteration and work towards the end of the string. To get the full picture, here's the `_match` helper method.

	protected static function _match($line, $number, $offset) {
		$string = substr($line, $offset);

		foreach(static::$_terminals as $pattern => $name) {
			if(preg_match($pattern, $string, $matches)) {
				return array(
					'match' => $matches[1],
					'token' => $name,
					'line' => $number+1
				);
			}
		}

		return false;
	}

We use the `preg_match` method to check if one of our pattern matches the current string. If you look closely, you can see that all of our regexes start at the beginning of the line (`^`) and 
are enclosed (`()`) so we can find them exactly at the beginning and get the inner content. We also store the current line, because we need it in our parser and to display helpful error messages.

Let's run our lexer with some example input:

	$input = array('root -> Foo::bar');
	$result = Lexer::run($input);
	var_dump($result);

We should get the following output:

	array
	  0 => 
		array
		  'match' => string 'root' (length=4)
		  'token' => string 'T_ROOT' (length=6)
		  'line' => int 1
	  1 => 
		array
		  'match' => string ' ' (length=1)
		  'token' => string 'T_WHITESPACE' (length=12)
		  'line' => int 1
	  2 => 
		array
		  'match' => string '->' (length=2)
		  'token' => string 'T_BLOCKSTART' (length=12)
		  'line' => int 1
	  3 => 
		array
		  'match' => string ' ' (length=1)
		  'token' => string 'T_WHITESPACE' (length=12)
		  'line' => int 1
	  4 => 
		array
		  'match' => string 'Foo' (length=3)
		  'token' => string 'T_IDENTIFIER' (length=12)
		  'line' => int 1
	  5 => 
		array
		  'match' => string '::' (length=2)
		  'token' => string 'T_DOUBLESEPARATOR' (length=17)
		  'line' => int 1
	  6 => 
		array
		  'match' => string 'bar' (length=3)
		  'token' => string 'T_IDENTIFIER' (length=12)
		  'line' => int 1

We got a stream of tokens that represents our source file. Let's run it with invalid input:

	$input = array('root -> ?invalid');
	$result = Lexer::run($input);

This raises an exception with the message `Unable to parse line 1.`.

# Wrapping Up
One thing you need to be aware of is that you need to place your token regex in the correct order - namely from special to general. If we put the `T_IDENTIFIER` before `T_ROOT`, the `root` keyword would always be matched as an identifier. While I haven't tried it out yet,[Nikita Popov](https://twitter.com/#!/nikita_ppv) [points out](http://nikic.github.com/2011/10/23/Improving-lexing-performance-in-PHP.html) that compiling all regexes into a big one leads to better performance.

Writing a simple lexer is not hard and (in my opinion) very flexible. If you need more  in your DSL, just add new regex <-> token mappings. I'd recommend to start with a small amount of tokens and writing unit-tests for each of them. If you add more tokens, add more unit tests, so you can be sure that your rules don't override each other and everything still works as expected.

As always, if you have feedback, ideas or improvements, let me know!