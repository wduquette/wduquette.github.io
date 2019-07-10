---
layout: post
title: The MoltResult Type
---

Rust provides the standard `Result<T,E>` type for returning values from functions or methods
that might need to include error info or the like.  C has no such thing; and so the C
implementation of Tcl has to jump through some hoops.  I'm going to describe how Standard Tcl
does it, and then how I've implemented the same pattern in Rust.

## How Tcl's C API Handles Results

In standard Tcl's C API, functions that might affect the flow of control return a _result code_:
an integer that indicates what happened.  The set of C functions cooperate to make the flow of
control indicated by the user's code actually happen.

There are five standard return codes:

* `TCL_OK`
* `TCL_ERROR`
* `TCL_BREAK`
* `TCL_CONTINUE`
* `TCL_RETURN`

The code `TCL_OK` indicates that the called function executed normally, and the caller should
continue executing normally.  

The code `TCL_ERROR` indicates that the function detected an error. The caller should (unless
it wishes to catch the error) just return `TCL_ERROR` itself, propagating the error upward.

The codes `TCL_BREAK` and `TCL_CONTINUE` are used by Tcl's loop control structures. They are
produced by the Tcl `break` and `continue` commands, which as you might expect are used to
break loop execution or jump to the beginning of the next loop iteration.  The caller should
propagate them upward—unless it's the function implementing the loop, in which case it should
catch and handle them.

The code `TCL_RETURN` is similar; it's produced by Tcl's `return` command, which is used to
return from Tcl procedures.  As with `TCL_BREAK` and `TCL_CONTINUE`, it propagates upward
until it is handled by the C function that executes Tcl procedures.

Thus, you see a lot of C code that looks like this;

```c
int result_code = Tcl_SomeFunction(interp, /* some args*/);

if (result_code != TCL_OK) {
    return result_code;
}
```

or

```c
char* value = Tcl_FindSomethingOrOther(interp, /* some args */);

if (value == null) {
    return TCL_ERROR;
}
```

But what about the Tcl procedure's computed return value?  In the `TCL_ERROR` case, just shown,
what about the error information?

The codes `TCL_OK` and `TCL_RETURN` always include a computed value (which may be the empty
string); and `TCL_ERROR` always includes an error message and some other data.  These things are
given to the `Tcl_Interp` object, represented above by the `interp` argument, as the interpreter's
_result_.  The `Tcl_Interp` class has a detailed API for setting, manipulating, and retrieving
the _result_.

## Rustifying Tcl Results

The above pattern should look pretty familiar.  It's basically the `Result<T,E>` pattern.  The
integer result codes take the place of `Ok` and `Err`, and the additional data passed via the
Tcl `interp` take the place of the `Ok` and `Err` tuple data.  Oh, and a whole lot of `if`
statements stand in for Rust's `?` operator.

Molt does it like this:

```rust
pub type MoltResult = Result<Value, ResultCode>;

#[derive(Eq, PartialEq, Clone, Debug)]
pub enum ResultCode {
    Error(Value),
    Return(Value),
    Break,
    Continue,
}
```

The `Value` type is Molt's standard representation for a data value, i.e., something that can be
put in a Molt variable.  I plan to write a post about `Value`, but for now if you think of it
as an immutable string you won't go far wrong.

A method or function in Molt's Rust API that computes a Molt `Value` returns `MoltResult`.
`Ok(Value)` handles the normal case, and corresponds to `TCL_OK`.  The four `ResultCode`
case correspond to the other Standard Tcl return codes, but don't require the interpreter's
help in passing back values.  Instead of `TCL_ERROR` we have `Err(ResultCode::Error(Value))`;
the `Value` is the error message.  Instead of `TCL_RETURN` we have
`Err(ResultCode::Return(Value))`, where the `Value` is the returned result.

In cases where we are computing something other than a `Value`, say an `i64`, we use that
directly, and return `Result<i64,ResultCode>`.

Consequently, we end up writing code like this:

```rust
let value: Value = interp.eval("some Molt code")?;

let index:i64 = molt_find_something(/* some args*)?;
...
```

Note that the Molt interpreter need not be involved!

## Tcl Liens

There are still a few things that the Standard Tcl implementation can do that Molt's
cannot.  

At present, on error Molt only returns an error message.  Tcl also saves an error code and
accumulates a stack trace as the stack unwinds.  The `ResultCode::Error` case needs to be
updated to handle that.

Also, Tcl extensions written in C do not need to confine themselves to the five canonical
result codes.  An extension that implements application-specific control structures, for example,
could use additional integer codes for own its own use.  That's more difficult with Molt's
architecture; I'd need to add an additional case to the `ResultCode` enum to support it.
