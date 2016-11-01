+++
date = "2016-10-28"
draft = false
title = "Text Analysis in Rust - Tokenization"
description = "This blog post shows you how to tokenize full text strings with rust efficiently."
tags = ["rust", "text", "indexing", "analysis"]
slug = "Text-Analysis-in-Rust-Tokenization"
+++

I work for [Couchbase](http://www.couchbase.com/) where we are currently developing full text search capabilities based on [bleve](http://blevesearch.com/). Bleve is implemented in [go](http://golang.org/) and inspired by [Apache Lucene](http://lucene.apache.org/core/), the reference implementation when it comes to full text search. While I am not directly involved in developing bleve I was curious about how it works internally and did take a look at the analyzers it provides.

Analyzers take your free form text and turn it into tokens which can then be used for indexing or queries. Since I know our team is currently at work doing performance optimizations in all places I wanted to take a stab at implementing similar analyzers in [Rust](https://www.rust-lang.org/) and see how they perform in comparison.

Analyzers typically consist of a tokenizer which slices a string into tokens and zero or more transformers which modify them subsequently (changing them, dropping them,...). A very common use case is to take some text, tokenize ("split") it by whitespace and then afterwards lowercase all tokens. This blog post only covers tokenizers, but transformers are planned for another post.

If you are only interested in how Rust is **3 times as fast** as the go equivalent in our microbenchmarks, check out the last section. If you are curious about the implementation details, read on.

When I think about tokenizers and transformers the first thing that comes to mind in Rust are [Iterators](https://doc.rust-lang.org/book/iterators.html). If we let our tokenizer implement the [Iterator](https://doc.rust-lang.org/beta/std/iter/trait.Iterator.html) trait we can make use of all the flexibility and high-level programming Rust provides without compromising performance.

## Tokenizers and Tokens
So lets define our `Tokenizer` as a trait which emits an `Iterator` of `Tokens`:

```rust
pub trait Tokenizer<'a> {
    /// A Tokenizer always needs to produce an Iterator of Tokens.
    type TokenIter: Iterator<Item = Token>;

    /// Takes the input string and tokenizes it based on the implementations rules.
    fn tokenize(&self, input: &'a str) -> Self::TokenIter;
}
```

Note: I can't wait for [impl Traits](https://github.com/rust-lang/rust/issues/34511) to land when it also supports trait functions, then this would clean up the declarations even further.

Lets take a first stab at the `Token`:

```rust
pub struct Token {
    term: String,
    start_offset: usize,
    position: usize,
}
```

The `Token` contains the `term` sliced by the tokenizer and also related metadata like its absolute start offset in bytes as well as the relative position of the token in the stream. We need this information down the road for indexing and querying.

One problem with this approach is that the `String` implicitly performs a heap allocation, and when we are slicing hundreds or thousands of small tokens we are allocating lots of small chunks on the heap.

The first idea to avoid this is using a `&str` instead, but we potentially need to modify the term and then we are back at heap allocations (as an exercise for the reader or as a future blog post, using [Cow](https://doc.rust-lang.org/std/borrow/enum.Cow.html) could also be an option). So, how can we make this more efficient?

One thing I realized during experimentation and later researching on the web is that when you are tokenizing text you mostly end up with lots of small tokens which maybe are even filtered out in the next step of the process. It would be nice if could stack allocate those small terms and as a fallback perform heap allocation for the rest! It turns out we can do just that, and here is one way:

```rust
const MAX_STACK_TERM_LEN: usize = 15;

enum Term {
    Stack(ArrayString<[u8; MAX_STACK_TERM_LEN]>),
    Heap(String),
}

pub struct Token {
    term: Term,
    start_offset: usize,
    position: usize,
}
```

Woops, our `Token` just got a little more complex. Our term is now an [Enum](https://doc.rust-lang.org/book/enums.html) which is either a stack allocated [ArrayString](https://docs.rs/arrayvec/0.3.20/arrayvec/struct.ArrayString.html) or a heap allocated [String](https://doc.rust-lang.org/std/string/struct.String.html). Note that the `ArrayString` implementation comes from the [arrayvec](https://crates.io/crates/arrayvec) crate and basically provides string functionality on top of a stack allocated array.

The nice thing about this is that we are not leaking this implementation detail to the caller, since we can always work with string slices on creation and access:

```rust
impl Token {
    #[inline]
    pub fn from_str(term: &str, start_offset: usize, position: usize) -> Self {
        Token {
            term: Token::convert_term(term),
            start_offset: start_offset,
            position: position,
        }
    }

    #[inline]
    fn convert_term(term: &str) -> Term {
        if term.len() <= MAX_STACK_TERM_LEN {
            Term::Stack(ArrayString::<[_; MAX_STACK_TERM_LEN]>::from(term).unwrap())
        } else {
            Term::Heap(term.to_string())
        }
    }

    #[inline]
    pub fn term(&self) -> &str {
        match self.term {
            Term::Heap(ref s) => s.as_ref(),
            Term::Stack(ref s) => s.as_ref(),
        }
    }
}
```

At this point we've departed from our go counterpart in two significant ways: we are using an iterator instead of [operating on a heap allocated array of tokens](https://github.com/blevesearch/bleve/blob/master/analysis/type.go#L61-L67) and we are using stack allocations for small tokens instead of heap allocating a [Token](https://github.com/blevesearch/bleve/blob/master/analysis/type.go#L41) and its term. I don't know if go is doing escape analysis and optimizations in this case, but based on the benchmarks (read on) I don't think so.

You might wonder why I selected a stack term length of 15 as the point where we start to do heap allocations. There is no fancy math behind it, I just tried tokenizing all kinds of text snippets and found this to be the sweet spot. Try it out for yourself, you'll see that when you are increasing the size by a larger number the whole tokenization gets slower since rust needs to move those huge chunks around on the stack. Oh, and I did pick 15 and not 16 since the `ArrayString` needs one byte for internal storage. If you get different numbers I'd love to read about it in the comments!

With all that in place, lets go work on some tokenizers.

## The Whitespace & Character Tokenizers
A common case in text analysis is splitting on whitespace, but thinking about more generally we probably want to split on all kinds of characters. So lets implement a character tokenizer which takes a function that takes the character as input and decides if it should split or not.

A quick warning: from here on it gets a bit more complex, so if you just came here for the benchmarks and you don't care about the actual implementation you can safely skip to the next section.

Here is our `CharTokenIter` struct:

```rust
pub struct CharTokenIter<'a> {
    filter: fn(&(usize, (usize, char))) -> bool,
    input: &'a str,
    byte_offset: usize,
    char_offset: usize,
    position: usize,
}
```

It takes a `filter` which is the function the implementations set when the `CharTokenIter` is created. Naturally it also stores a reference to the original `input` since it needs to create the tokens from it. Finally three variables are used to keep internal state that we'll see getting populated in a second. `byte_offset` stores the absolute offset in bytes of the token term, the `char_offset` does the same but it only counts full characters. Finally the `position` stores the relative position of the tokens in the stream. If you wonder why we are handling characters and bytes separately, welcome to the world of [Unicode](https://en.wikipedia.org/wiki/Unicode).

We can make creating our iterator a bit nicer:

```rust
impl<'a> CharTokenIter<'a> {
    pub fn new(filter: fn(&(usize, (usize, char))) -> bool, input: &'a str) -> Self {
        CharTokenIter { filter: filter, input: input, byte_offset: 0, char_offset: 0, position: 0 }
    }
}
```

Now, lets get to where the rubber meets the road. We need to implement the `Iterator` as specified in the contract, so our function looks like this:

```rust
impl<'a> Iterator for CharTokenIter<'a> {
    type Item = Token;

    fn next(&mut self) -> Option<Token> {
    	// ponies and fairy dust in here...
    }
}
```

The rest of the code snippets that follow go inside the `next` function. If we want to satisfy the Iterator semantics, we need to emit a token every time the predicate provided by our `filter` matches:

```rust
 for (cidx, (bidx, c)) in self.input[self.byte_offset..]
            .char_indices()
            .enumerate()
            .filter(&self.filter) {
}
```

We are slicing the input string starting at the current `byte_offset`, then use `char_indices()` on the iterator to also get the character index and then use `enumerate()` to get the byte index. Equipped with the actual character and all kinds of offsets, we apply our filter function and if allowed to pass through craft our token and emit it:


```rust
let slice = &self.input[self.byte_offset..self.byte_offset + bidx - skipped_bytes];
let token = Token::from_str(slice, self.char_offset, self.position);
self.char_offset = self.char_offset + slice.chars().count() + 1;
self.position += 1;
self.byte_offset = self.byte_offset + bidx + char_len - skipped_bytes;
return Some(token);
```

You can see that we are not only slicing out the token term but also adjusting the character and byte offsets properly. Now, what is this `skipped_bytes` variable about? We'll get to that in a second, but before that we need to cover another case: what happens if no split character is found or there are characters left after the last split character? To handle this, after our loop we need to handle the remainder:

```rust
if self.byte_offset < self.input.len() {
    let slice = &self.input[self.byte_offset..];
    let token = Token::from_str(slice, self.char_offset, self.position);
    self.byte_offset = self.input.len();
    Some(token)
} else {
    None
}
```

If our byte offset is equal to the length of the input string we are done (signaled by returning `None` to the iterator consumer), but in the other case we need to slice out the remaining bytes and return them as a final token.

Now that we've covered this lets get back to the `skipped_bytes` thing from before.

We need to handle one more case: it can happen that more than one filter characters appear right after one another or at the very beginning of the text. Consider splitting on whitespace and the input looks like `  hello \t\n world!  `. You want two tokens `hello` and `world!` and all the other whitespace should be consumed (accounted for in the byte offset, but not emitted inside an actual token term). To handle this case we keep track of the characters inside our for loop before emitting the token:

```rust
// Temp variables to account skipping
let mut skipped_bytes = 0;
let mut skipped_chars = 0;
for (cidx, (bidx, c)) in self.input[self.byte_offset..]
    .char_indices()
    .enumerate()
    .filter(&self.filter) {
        let char_len = c.len_utf8();

        // Check and skip chars if needed
        if cidx - skipped_chars == 0 {
            self.byte_offset = self.byte_offset + char_len;
            self.char_offset += 1;
            skipped_bytes = skipped_bytes + char_len;
            skipped_chars += 1;
            continue;
        }

        // Same as before down here
        let slice = &self.input[self.byte_offset..self.byte_offset + bidx - skipped_bytes];
        let token = Token::from_str(slice, self.char_offset, self.position);
        self.char_offset = self.char_offset + slice.chars().count() + 1;
        self.position += 1;
        self.byte_offset = self.byte_offset + bidx + char_len - skipped_bytes;
        return Some(token);
    }
```

Here is the `next` function in its full glory:

```rust
fn next(&mut self) -> Option<Token> {
    let mut skipped_bytes = 0;
    let mut skipped_chars = 0;
    for (cidx, (bidx, c)) in self.input[self.byte_offset..]
        .char_indices()
        .enumerate()
        .filter(&self.filter)
        {
            let char_len = c.len_utf8();
            if cidx - skipped_chars == 0 {
                self.byte_offset = self.byte_offset + char_len;
                self.char_offset += 1;
                skipped_bytes = skipped_bytes + char_len;
                skipped_chars += 1;
                continue;
            }

            let slice = &self.input[self.byte_offset..self.byte_offset + bidx - skipped_bytes];
            let token = Token::from_str(slice, self.char_offset, self.position);
            self.char_offset = self.char_offset + slice.chars().count() + 1;
            self.position += 1;
            self.byte_offset = self.byte_offset + bidx + char_len - skipped_bytes;
            return Some(token);
        }

    if self.byte_offset < self.input.len() {
        let slice = &self.input[self.byte_offset..];
        let token = Token::from_str(slice, self.char_offset, self.position);
        self.byte_offset = self.input.len();
        Some(token)
    } else {
        None
    }
}
```

How would we implement a `Whitespace` tokenizer with this? One might be tempted to match against `' '`, but keep in mind that whitespace, when dealing with unicode, can be many more different characters so its best to not hardcode them in a gigantic match clause and instead use a built-in function.

Here is our `WhitespaceTokenizer`:

```rust
pub struct WhitespaceTokenizer;

impl<'a> Tokenizer<'a> for WhitespaceTokenizer {
    type TokenIter = CharTokenIter<'a>;

    fn tokenize(&self, input: &'a str) -> Self::TokenIter {
        CharTokenIter::new(is_whitespace, input)
    }
}

#[inline]
fn is_whitespace(input: &(usize, (usize, char))) -> bool {
    let (_, (_, c)) = *input;
    c.is_whitespace()
}
```

We initialize the `CharTokenIter` and in use the [char::is_whitespace()](https://doc.rust-lang.org/std/primitive.char.html#method.is_whitespace) method to check for a unicode whitespace character. Using it is now pretty simple, consider this test case:

```rust
#[test]
fn should_split_between_words() {
    let expected = vec![Token::from_str("hello", 0, 0), Token::from_str("world", 6, 1)];
    let actually = WhitespaceTokenizer.tokenize("hello world").collect::<Vec<Token>>();
    assert_eq!(expected, actually);
}
```

This implementation also works with all kinds of unicode characters that may have different byte lengths (and this is where our different tracking of character and byte offsets becomes important):

```rust
#[test]
fn should_handle_mixed_chars() {
    let expected = vec![Token::from_str("abc界", 0, 0), Token::from_str("abc界", 5, 1)];
    let actually = WhitespaceTokenizer.tokenize("abc界 abc界").collect::<Vec<Token>>();
    assert_eq!(expected, actually);
}
```

Before we move on to benchmarks, lets take a quick look at the corresponding [go implementation](https://github.com/blevesearch/bleve/blob/master/analysis/tokenizer/character/character.go).

The [CharacterTokenizer](https://github.com/blevesearch/bleve/blob/master/analysis/tokenizer/character/character.go#L35) loops through each [rune](https://blog.golang.org/strings) and very similar to our implementation keeps track of all kinds of offsets and slices out terms as needed. The big difference is that it can't do explicit stack allocations and upfront allocates a 1024 element slice for its `TokenStream` which would be extended if more tokens are needed.

The [WhitespaceTokenizer](https://github.com/blevesearch/bleve/blob/master/analysis/tokenizer/whitespace/whitespace.go#L32) is also very similar and calls a runtime-provided function that checks if the given rune is a unicode whitespace character or not.

## Benchmarks
This is what you came for, right? Lets see how our implementation performs in comparison to the go one. To establish a meaningful baseline we need to define a common text corpus that is used for the benchmark.

**Edit:** I forgot to mention the versions used: Rust is at `1.14.0-nightly` since the benchmarking tool is unstable and the version of go used is ` go1.7.1 darwin/amd64`.

I copied the text out of the [Rust Wikipedia](https://en.wikipedia.org/wiki/Rust_(programming_language)) page that is exact 1000 characters long and contains around 150-160 tokens depending on the type of tokenizer used.

```rust
static INPUT: &'static str =
    "In addition to conventional static typing, before version 0.4, Rust also supported \
     typestates. The typestate system modeled assertions before and after program statements, \
     through use of a special check statement. Discrepancies could be discovered at compile time, \
     rather than when a program was running, as might be the case with assertions in C or C++ \
     code. The typestate concept was not unique to Rust, as it was first introduced in the \
     language NIL. Typestates were removed because in practice they found little use, though the \
     same functionality can still be achieved with branding patterns.

The style changed between \
     0.2, 0.3 and 0.4. Version 0.2 introduced classes for the first time, with version 0.3 adding \
     a number of features including destructors and polymorphism through the use of interfaces. \
     In Rust 0.4, traits were added as a means to provide inheritance; In January 2014, the \
     editor-in-chief of Dr Dobb's, Andrew Binstock, commented on Rust's chances to become a \
     competitor to C++.";
```

If we split on unicode whitespace, there are exactly 159 tokens.

Here is our benchmark for rust:

```rust
#[bench]
fn bench_whitespace_tokenizer(b: &mut Bencher) {
    let t = WhitespaceTokenizer;
   b.iter(|| t.tokenize(INPUT).last());
}
```

Running with `cargo +nightly bench` on my machine (a 2,5GHz i7, OSX, 16GB RAM), rust reports:

```
test bench_whitespace_tokenizer ... bench:       6,972 ns/iter (+/- 1,148)
```

It takes 7 microseconds to process 159 tokens, so it can process 23 tokens per microsecond. This translates into roughly 23 million tokens per second. Put differently, with this input text we can tokenize around 136MB/s on a single core.

Here is our go equivalent in bleve:

```go
var sampleLargeInput = []byte(`In addition to conventional static typing, before version 0.4, Rust also supported \
     typestates. The typestate system modeled assertions before and after program statements, \
     through use of a special check statement. Discrepancies could be discovered at compile time, \
     rather than when a program was running, as might be the case with assertions in C or C++ \
     code. The typestate concept was not unique to Rust, as it was first introduced in the \
     language NIL. Typestates were removed because in practice they found little use, though the \
     same functionality can still be achieved with branding patterns.

The style changed between \
     0.2, 0.3 and 0.4. Version 0.2 introduced classes for the first time, with version 0.3 adding \
     a number of features including destructors and polymorphism through the use of interfaces. \
     In Rust 0.4, traits were added as a means to provide inheritance; In January 2014, the \
     editor-in-chief of Dr Dobb's, Andrew Binstock, commented on Rust's chances to become a \
     competitor to C++.`)

func BenchmarkTokenizeEnglishText(b *testing.B) {

	tokenizer := character.NewCharacterTokenizer(notSpace)
	b.ResetTimer()

	for i := 0; i < b.N; i++ {
		tokenizer.Tokenize(sampleLargeInput)
	}

}
```

It reports:

```
BenchmarkTokenizeEnglishText-8   	   50000	     22893 ns/op	   19072 B/op	     171 allocs/op
```

So each run takes close to 23 microseconds compared of 7 in Rust! That's roughly three times slower, clocking at
around 7 million tokens per second or 41MB/s per core.

Why is that? Let's try disabling the stack allocation optimization in rust by setting the `MAX_STACK_TERM_LEN` to 0 and rerun the benchmark:

```
test bench_whitespace_tokenizer ... bench:      10,154 ns/iter (+/- 1,553)
```

We lost a couple microseconds here, but I think the rest is just LLVM and Rust being really good at optimizing the iterator and loop code (keep in mind we are still stack-allocating the tokens themselves while in go I'm pretty sure its heap allocated as well in the big `TokenStream` slice).

Now with any kind of microbenchmarking it is hard to say how the difference will look like on your hardware and dataset. More realistic numbers could be gathered by tokenizing large chunks of for example the Wikipedia dataset and then looking at the overall numbers. Of course your mileage may also vary per machine, so if you are curious please run the benchmarks on your hardware - and please post them in the comments below!
