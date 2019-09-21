---
layout: post
title: Molt 0.1.1 Published to Crates.IO
---

I've just published [Molt 0.1.1](https://github.com/wduquette/molt) to Crates.IO.  The system
includes:

**molt 0.1.1**: The language interpreter.  This is the core of Molt, and is intentionally
design to have no external dependencies (other than Rust and its standard libraries.)  At some
point, this crate may get updated to support "no_std".  If you simply want to embed the
Molt interpreter in an application, this is all you need.

**molt-shell 0.1.1**: An application framework for Molt REPLs, test harnesses, and
benchmarks.  The REPL provides command editing and history via the `rustyline` crate.  This
crate is intended for actual use, and also as an example.

**molt-app 0.1.1**: A plain vanilla `moltsh` application implemented using `molt-shell`. It
provides the REPL, test harness, and benchmark harness, and is the primary tool for
development of Molt itself.  Most users of Molt will want to define their own applications
so that they can extend Molt with their own application-specific commands.

**molt-sample**: The [`molt-sample` repo](https://github.com/wduquette/molt-sample) contains
a sample Molt extension crate.  It shows how to:

*   Create a custom Molt REPL using `molt-shell`, and extend it with a new Molt command.
*   How to define a Rust library crate that can install new Molt commands into a Molt
    `Interp`.  The library crate can define commands in both Rust and TCL.
