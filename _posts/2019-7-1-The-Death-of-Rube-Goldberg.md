---
layout: post
title: The Death of Rube Goldberg
---

I began learning the [Rust programming language](https://www.rust-lang.org) late in 2018, and as an old C
programmer I was immediately impressed by how much simpler and easier it is to configure and
build a cross-platform project in Rust than it is in C. I called this post
"The Death of Rube Goldberg" because tools like `cargo`, the Rust build
manager, are ultimately going to kill the old C ecosystem of `autoconf`, `configure`, `make`,
_et al_.

Rust is a lovely language; but my great love is [Tcl](https://www.tcl-lang.org).  Tcl got its start as an
extension language for C; and no sooner had I begun to work with Rust than I kinda started to
want to be able to extend it in Tcl.  Shortly thereafter I began working on
[Molt](https://github.com/wduquette/molt), a Tcl interpreter written in Rust rather than C.  
I plan to use this blog to talk about Tcl and Rust, and especially about Molt and its
implementation.

<!--
![_config.yml]({{ site.baseurl }}/images/config.png)

-->
