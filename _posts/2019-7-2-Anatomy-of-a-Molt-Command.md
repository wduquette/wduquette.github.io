---
layout: post
title: Anatomy of a Molt Command
---

Tcl is a sort of a cross between shell languages and LISP, with pretensions to C syntax.  As such,
the language consists of a collection of _commands_.  A statement in the language is also called
a _command_, and consists of one or more white-space delimited words.  For example, the following
command appends the string `"Some more text"` the variable `buffer`:

```tcl
append buffer "Some more text"
```

The `append` command can append any number of values to the variable.  The following command
is equivalent to the first.  

```tcl
append buffer "Some " more " text"
```

Also, notice that strings in Tcl don't actually need to be quoted unless they
contain whitespace.  That's because in Tcl, **Everything is a string.**  It's a dynamic
language, and so any value can be used in any way that's consistent with its string
representation.  The number "5" can be an integer, a string, or a list of one element.

In Tcl 8.6, the `append` command is defined as follows.  (I include the entire function
for reference; I'll be calling attention to specific features further down.)

```c
int
Tcl_AppendObjCmd(
    ClientData dummy,           /* Not used. */
    Tcl_Interp *interp,         /* Current interpreter. */
    int objc,                   /* Number of arguments. */
    Tcl_Obj *const objv[])      /* Argument objects. */
{
    Var *varPtr, *arrayPtr;
    register Tcl_Obj *varValuePtr = NULL;
                                /* Initialized to avoid compiler warning. */
    int i;

    if (objc < 2) {
        Tcl_WrongNumArgs(interp, 1, objv, "varName ?value ...?");
        return TCL_ERROR;
    }

    if (objc == 2) {
        varValuePtr = Tcl_ObjGetVar2(interp, objv[1], NULL,TCL_LEAVE_ERR_MSG);
        if (varValuePtr == NULL) {
            return TCL_ERROR;
        }
    } else {
        varPtr = TclObjLookupVarEx(interp, objv[1], NULL, TCL_LEAVE_ERR_MSG,
                "set", /*createPart1*/ 1, /*createPart2*/ 1, &arrayPtr);
        if (varPtr == NULL) {
            return TCL_ERROR;
        }
        for (i=2 ; i<objc ; i++) {
            /*
             * Note that we do not need to increase the refCount of the Var
             * pointers: should a trace delete the variable, the return value
             * of TclPtrSetVarIdx will be NULL or emptyObjPtr, and we will not
             * access the variable again.
             */

            varValuePtr = TclPtrSetVarIdx(interp, varPtr, arrayPtr, objv[1],
                    NULL, objv[i], TCL_APPEND_VALUE|TCL_LEAVE_ERR_MSG, -1);
            if ((varValuePtr == NULL) ||
                    (varValuePtr == ((Interp *) interp)->emptyObjPtr)) {
                return TCL_ERROR;
            }
        }
    }
    Tcl_SetObjResult(interp, varValuePtr);
    return TCL_OK;
}
```

The Molt equivalent looks like this:

```rust
pub fn cmd_append(interp: &mut Interp, argv: &[Value]) -> MoltResult {
    check_args(1, argv, 2, 0, "varName ?value value ...?")?;

    // FIRST, get the value of the variable.  If the variable is undefined,
    // start with the empty string.

    let var_name = &*argv[1].as_string();
    let var_result = interp.var(var_name);

    let mut new_string = if var_result.is_ok() {
        // Use to_string() because we need a mutable owned string.
        var_result.unwrap().to_string()
    } else {
        String::new()
    };

    // NEXT, append the remaining values to the string.
    for item in &argv[2..] {
        new_string.push_str(&*item.as_string());
    }

    // NEXT, save and return the new value.
    molt_ok!(interp.set_var2(var_name, new_string.into()))
}
```

First, observe the function signatures:

```c
int
Tcl_AppendObjCmd(ClientData dummy, Tcl_Interp *interp, int objc, Tcl_Obj *const objv[]) {
    ...
}
```

```rust
pub fn cmd_append(interp: &mut Interp, argv: &[Value]) -> MoltResult {
    ...
}
```

In both cases the function takes the interpreter and an array containing the command and
its arguments.  In Tcl 7.6 and earlier, the array would have been any array of strings; in
Tcl 8.6 and Molt, it's a special type (`Tcl_Obj`, `Value`) an immutable string that may also
take on specific binary data representations.  (I'll talk about the anatomy of `Value` in
a later post.)

The `Tcl_AppendObjCommand` also takes a `ClientData` argument; this is used to associate
context data with the command.  It isn't needed for `append`.

Finally, notice the return value: `int` vs. `MoltResult`, which is a type alias:

```rust
pub type MoltResult = Result<Value, ResultCode>;
```

In Rust, the `append` command returns a `Value` on success and a `ResultCode` (which is very much more than a simple error) on failure.  We'll come back to that.

The next step is to validate the argument list.  The `append` command must take at least two words, the command name and a variable name.  If it contains only one word, that's an error.
The C code handles this by calling `Tcl_WrongNumArgs` to generate an error message, and then
returns the integer code `TCL_ERROR`.  The error message gets stashed in the interpreter's  "result" field.

```c
if (objc < 2) {
     Tcl_WrongNumArgs(interp, 1, objv, "varName ?value ...?");
     return TCL_ERROR;
}
```

The error message looks like this (question marks in Tcl command signatures indicate optional arguments):

```
wrong # args: should be "append varName ?value value ...?"
```

The Molt code does exactly the same thing, but much more concisely.  The `check_args` method checks the minimum and maximum number of arguments against the actual number; if there are two
few it produces the same error message and returns it as part of the `MoltResult`.  

```rust
check_args(1, argv, 2, 0, "varName ?value value ...?")?;
```


Interestingly, the pattern is exactly the same; the only significant difference is that Rust provides first class support for it in the language itself via the `Result<T,E>` type and the
`?` operator.

Once the new string has been created, it's assigned back to the variable; and then new string is returned to the caller. In C the value is returned by explicitly setting the interpreter's result field, and the integer constant `TCL_OK` is returned to indicate that the command
executed successfully.

```c
Tcl_SetObjResult(interp, varValuePtr);
return TCL_OK;
```

The constant `TCL_OK` indicates that the command executed successfully.  

In Rust, we use the following code, which assigns the new string to the variable, gets it back as a `Value`, and wraps it up in `Ok()`:

```rust
molt_ok!(interp.set_var2(var_name, new_string.into()))
```

The `molt_ok!` macro takes an argument or arguments in a number of forms, converts its
arguments to a `Value` if necessary, and wraps the value in `Ok()`.  We could also write it like this:

```rust
let value = interp.set_var2(var_name, new_string.into());
Ok(value)
```

The Rust version is shorter, and undeniably easier to read.  (And I should add that the Tcl code base is considered quite readable—for C code.)

## Coming Attractions

* The Molt `ResultCode`
* The Molt `Value` Type
* Commands with Context
