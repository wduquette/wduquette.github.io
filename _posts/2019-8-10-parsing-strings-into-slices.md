---
layout: post
title: Parsing Rust Strings into Slices
---

A Rust `String` is a vector of bytes containing a UTF-8 string, which is an uneasy
combination.  You can't simply index into a `String`: the compiler forbids it, because you're
far too likely to get one byte of a multi-byte UTF-8 `char`.  Instead you need to use a `Chars`
iterator to parse out the string character by character.

```rust
// A string to play with; ironically, one that's entirely ASCII.
let source = "abcdefghijklmnopqrstuvwxyz";

let mut c1 = source.chars();

while let Some(ch) = c1.next() {
    // Do something with the characters
}
```

## The Naive Way

Now, suppose you're writing a parser.  You want to build a syntax tree, or something of the sort;
and for efficiency you want the tree to contain slices of the source string.  You're pulling
individual characters out of the source string; how do you accumulate them into slices?

For a newbie Rust programmer, the solution is Not Obvious.  To get started, I resorted to doing
this sort of thing:

```rust
// Let's parse a number!
let source = "...";  // Some source string.

let mut c1 = source.chars();
let mut number: String::new();

while let Some(ch) = c1.next() {
    // Do something with the characters
    if ch.is_digit(10) {
        number.push(ch);
    } else {
        break;
    }
}

// number contains the parsed number, or is empty.
```

This gets the right answer, but it's slow.  In C, you'd just save a reference to the token: a
`char*` pointer and a length.  Here we're doing a whole bunch of allocation.

## The Better Way

The answer turns out to be surprisingly simple, and is based on two facts:

* The `Chars` iterator can return a `&str` slice containing the remainder of the source string.
* It's easy to compute a slice from two slices one of which completely contains the other.

For the record, I found the solution at
[users.rust-lang.org](https://users.rust-lang.org/t/takewhile-iterator-over-chars-to-string-slice/11014).

Consider this:

```rust
// Let's parse a number!
let source = "1234 abcd";

let mut c1 = source.chars();

while let Some(ch) = c1.next() {
    if !ch.is_digit(10) {
        break;
    }
}

// ch points at the first non-digit character.
// And the length of the desired slice in bytes is the
// length of the source string minus the length of the remainder.

// Yes, I really ought to check whether I got any characters.

let num: &str = &source[..source.len() - c1.as_str().len()];

// num is a slice containing the desired number.
assert_eq!(num, "1234");
assert_eq!(c1.as_str(), " abcd");
```

## Wrapping it Up

Of course, you'd want to wrap that up in a function so you don't need to think about it so
much.  Here's one possibility.

```rust
/// Given a string slice and an iterator on that slice, return
/// a slice of the text from the beginning of the slice up
/// to the iterator.
pub fn get_slice<'a>(start: &'a str, end: &Chars) -> &'a str {
    &start[..start.len() - end.as_str().len()]
}
```

## But what if you need to `peek()`?

There's a problem with using `Chars::as_str` in this way: if you need lookahead.  

```rust
let source = "1234";
assert_eq!(source.chars().as_str(), source);

let ps = source.chars().peekable();

// You can't do this: Peekable<Chars> doesn't know anything
// about Chars except that it's an Iterator.
assert_eq!(ps.as_str(), source);
```

Once again, [users.rust-lang.org](https://users.rust-lang.org/t/losing-std-as-str/31262)
provides the answer: use `CharIndices`.

```rust
let source = "1234";
let mut c1 = source.char_indices().peekable();

...

// start is the starting index of the character.
let Some((start,ch)) = c1.peek();

// Some code that advances the iterator.
...
let Some((end,ch)) = c1.peek();

// Grab everything from the first character we peeked at up to
// but not including the last character we peeked at.
let token = &source[start..end];
```
