# A Rust syntax extension to make `match` nicer to use

## Notes

Discussion is on [reddit](http://www.reddit.com/r/rust/comments/1z8p11/an_idea_for_a_syntax_extension_to_make_match/).  This has been revised since I first submitted the reddit link, so some of the comments there might not make sense in context of this new version.  [Here is the original version from when it was first submitted.](https://github.com/MicahChalmer/rust-match-ext/blob/original/README.md)

I still haven't actually written the syntax extension yet.  All the expansions below were done by hand, in my head.  There are probably mistakes.  I put it up in public anyway, with just this README, in order to link to it from reddit and get some feedback on the design.  The aforementioned revision is the result of that feedback.

## Motivation

Oh, how we hate the `match` statement! Methods like `and_then` and `or_else` are added to `Option` to avoid having to use it.  The `try!` macro was added to avoid having to use `match` with `Result`.  Proposals are discussed daily on the mailing list and r/rust to try to get out of using it--monads, refutable let statements, etc.

Why is `match` so annoying to use?  The patterns are great, but as soon as you want to do more than a one-liner the variables you bind in a pattern, you run into two problems:

   1. You are forced to use _two_ levels of block indentation for each match statement
   2. When one match statement is nested in another, the match alternatives are distant from each other and in a confusing order overall.

A great illustration of both of these problems is this ["pyramid of doom" code snippet](https://github.com/nick29581/rust/commit/86c4055913c11e8e2d8fd830cf5d3a46c77ecd9a#src-librustc-metadata-encoder-rs-P19) that appeared in a [pull request](https://github.com/mozilla/rust/pull/12562):

````rust
let mut cur_struct = struct_def;
let mut structs: ~[@StructDef] = ~[];
loop {
    match cur_struct.super_struct {
        Some(t) => match t.node {
            ast::TyPath(_, _, path_id) => {
                let def_map = tcx.def_map.borrow();
                match def_map.get().find(&path_id) {
                    Some(&DefStruct(def_id)) => {
                        cur_struct = match tcx.map.find(def_id.node) {
                            Some(ast_map::NodeItem(i)) => {
                                match i.node {
                                    ast::ItemStruct(struct_def, _) => struct_def,
                                    _ => ecx.diag.handler().bug("Expected ItemStruct"),
                                }
                            },
                            _ => ecx.diag.handler().bug("Expected NodeItem"),
                        };
                    },
                    _ => ecx.diag.handler().bug("Expected DefStruct"),
                }
            }
            _ => ecx.diag.handler().bug("Expected TyPath"),
        },
        None => break,
    }
}
````

Quick--without tracing upwards with your finger or cursor to line up the indentation, which `exc.diag.handler().bug` message goes with which pattern?

The structures being matched are more than a bunch of Results and Options, so there's no way to `try!` or `map` around it.  The comments on the pull request suggested a macro, but that has to be custom-written in each case.  So here's my idea for a syntax extension that could address these problems in a general way. 

## Example

I would like to create a `match!` syntax extension such that the code below would expand into something equivalent to the pyramid snippet above:

````rust
let mut cur_struct = struct_def;
let mut structs: ~[@StructDef] = ~[];
loop {
    match! {
        let Some(t) match cur_struct.super_struct else break;

        let ast::TyPath(_, _, path_id) match t.node else ecx.diag.handler().bug("Expected TyPath");

        let def_map match tcx.def_map.borrow();
        let Some(&DefStruct(def_id)) match def_map.get().find(&path_id) else
            ecx.diag.handler().bug("Expected DefStruct");

        cur_struct = match! {
            let Some(ast_map::NodeItem(i)) match tcx.map.find(def_id.node) else
                ecx.diag.handler().bug("Expected NodeItem");

            let ast::ItemStruct(struct_def, _) match i.node else
                ecx.diag.handler().bug("Expected ItemStruct");

            struct_def
        }
    };
}
````

I, for one, find that much easier to follow.  Next to each pattern, you can see the error used to escape if the pattern doesn't match.  You can read the whole thing in a sequence and understand the flow of the logic.

(Because the `bug` calls don't return, in this case the inner `match!` invocation could be eliminated, ending the whole thing with `cur_struct = struct_def` instead.  That's true of the original version as well, but there it's not so obvious--it's easy to get lost in all the nested match statements.)

## Features

A `match!` block is like an ordinary block--it contains a sequence of statements and evaluates to the result of the last statement.  But it gets additional features that can be used inside it:

   1. Refutable `let`
   2. Early return from the match with unary `^`

### Refutable `let`

The first feature: inside a `match!` you can use refutable patterns in `let` statements.  This idea was suggested [on the mailing list by Gabor Lehel](https://mail.mozilla.org/pipermail/rust-dev/2013-December/007480.html).  The idea is that you give a refutable pattern to bind your variables in the `let` statement, and add an `else` clause to say what happens if the binding pattern does not match.

````rust
let primary_pattern match expr else match_arm, match_arm...
````

Here a "match arm" is the same as in a normal match statement (`pattern [ if expr] => [ expr | block  ]`).  If the primary pattern matches, the block continues with the variables from the primary pattern in scope.  Otherwise, one of the match arms that come after the `else` are used.  The `else` arms cannot return anything (their type must be `!`), so they are constrained to end in `break`, `continue`, `return`, or calls that never return such as `fail!` or `libc::exit`.

The "refutable" refers only to the main binding pattern.  Overall, the combination of the binding pattern followed by all the `else` match arms must be exhaustive, otherwise it's a compile-time error.  There is no implicit posisbility of runtime failure here.

As an additional shortcut, if there is only one else arm whose pattern is `_` (match anything), the `_ =>` may be left out.  In other words: `let x match y else z` is equivalent to `let x match y else _ => z`.


So for example, this would be valid:

````rust
loop {
    match! {
        // here is some stuff...
        let Foo(f) match xyz else Bar => continue, Baz => return "got a baz";
        // ...and here is more stuff that uses f...
    }
}
````

and would expand into this:

````rust
loop {
    // here is some stuff...
    match xyz {
        Foo(f) => {
            // ...and here is more stuff that uses f...
        },
        Bar => continue,
        Baz => return "got a baz"
    }
}
````

I've kept the refutable `let` syntactically different than normal irrefutable `let` in that the `match` keyword is used instead of the `=`: `let REFUTABLE_PATTERN match EXPR else ALTERNATIVES` vs `let IRREFUTABLE_PATTERN = EXPR`.  That isn't strictly necessary--it could have been `let REFUTABLE_PATTERN = EXPR else ALTERNATIVES`.  But I think it's helpful to keep the distinction for now (and it will also probably make it easier to implement as an external syntax extension.)

There has been discussion in the [email list](https://mail.mozilla.org/pipermail/rust-dev/2013-December/007482.html) and [on reddit](http://www.reddit.com/r/rust/comments/1z8p11/an_idea_for_a_syntax_extension_to_make_match/) about letting the `else` arms return values, and using those to bind to the pattern variables.  But that adds a lot of complexity and I'm not sure it's all that useful, so for the time being I will simply say all the `else` arms must be of type `!`, which leaves the door open for expanding it later.


### Early return from `match!` blocks

If the `else` arm of a refutable `let` has to be of type `!`, how can we replace match pyramids that have to return values and are not the only statement in a function?  Suppose we have this:

````rust
let r = match a {
    B(b) => match get_c(b) {
        C(c) => match get_d(c) {
            D(d) => {
                // do stuff with d...
            }
            _ => Err("wanted d")
        },
        _ => Err("wanted c")
    },
    _ => Err("wanted b")
}
do_something_with(r);  //Expects r as a Result<D,&'static str>
````

How can we write this without nesting, but still get the `Err` values into the result `r`?  To enable this, the `match!` syntax extension enables syntax to specify a value that is returned from the overall `match!` block immediately.  It's equivalent to what `break` does for loops, but takes an expression like `return`.  To keep it syntactically smaller and unambiguous, we use a unary `^` for this.  So the following would be equivalent to the pyramid above:

````rust
let r = match! {
    let B(b) match a else ^Err("wanted b");
    let C(c) match get_c(b) else ^Err("wanted c");
    let D(d) match get_d(c) else ^Err("wanted d");
    // do stuff with d...
}
do_something_with(r);  //Expects r as a Result<D,&'static str>
````

The unary `^` can be used anywhere inside a `match!` block, not just as alternative branches to refutable lets.  So this would be valid:

````rust
let z = match! {
    if foo() {
        ^ "foo was true";
    }
    // do more stuff...
    "foo was false"
}
````

## Possible extensions (that I'm not going to implement)

There are a few potential additions to the features described above.  I'm probably not going to try to implement them at first, since they'd make it more difficult to and I'm not sure they're really necessary.  But I'll describe them here anyway for the sake of discussion:

### Early match return with labels

If you nest `match!` blocks, there is no way to early-return from the outer block from inside the inner one.  The obvious way to allow this is to use labels in the same way that `break` does:

````rust
let outer:IoResult<Whatever> = 'outer: match! {
    let inner:Option<SomethingElse> = match! {
        let Ok(line) match io::stdin().read_line() else
            Err(IoError{kind:EndOfFile,..}) => ^None,
            Err(err) => ^'outer Err(err);  // Better/different syntax here maybe?
        // do stuff with the line...
        Some(get*something*else())
    }
    // do more stuff...
    Ok(get*whatever())
}
````

This exact syntax wouldn't be possible from an external syntax extension.  The parser wouldn't allow the label to come before anything that wasn't `loop`, `for` or `while`, and it checks for that before macro expansion, not after.  To do it from an external syntax extension would require the label to go somewhere inside the block, and that's even uglier.

Another problem is that the `^'scope expr` syntax has ambiguity problems, because closure expressions can start with a lifetime, but this is [hopefully about to change](https://github.com/mozilla/rust/issues/10553), so that shouldn't be an issue for much longer.

So this is something to add later if needed and desired.

### Default alternative branches

Here's an example of using a `match!` block instead of the `try!` macro.  It would have the advantage of not needing to return out of the entire fn:

````rust
let res:IoResult<Whatever> = match! {
    let Ok(p) match Process.new("proggy", [~"arg1", ~"arg2"]) else Err(err) => ^Err(err);
    let Some(stdin) match p.stdin  else ^Err(IoError{kind: ResourceUnavailable,
                                                     desc:"Process has no stdin", detail:None});

    let Ok(_) match writeln!(stdin, "Hello there!") else Err(err) => ^Err(err);
    let Some(stdout) match p.stdout ^ None=>Err(IoError{kind: ResourceUnavailable,
                                                        desc:"Process has no stdout", detail:None});
    let Ok(line) match BufferedStream::new(stdout).read_line() else Err(err) => ^Err(err);
    // more stuff...
}
// do something with res, gets here on errors as well as Ok
````

That's OK so far as it goes, but we keep repeating `else Err(err) => ^Err(err)` a lot.  (Just `else err => err` wouldn't work, because then the `err` pattern represents the whole `Result`, and the `Ok` type would have to match the outer type, not just the `Err` type.)  So maybe there could be a way to specify a default within each `match!` block.  A possible syntax:

````rust
match! {
    match else Err(err) => ^Err(err);  // declare the default non-primary match.  In effect from where it appears to the end of the enclosing pair of curly braces
    let Ok(p) match Process.new("proggy", [~"arg1", ~"arg2"]);
    let Some(stdin) match p.stdin else ^Err(IoError{kind: ResourceUnavailable,
                                                    desc:"Process has no stdin", detail:None});

    let Ok(_) match writeln!(stdin, "Hello there!");
    let Some(stdout) match p.stdout else None=>^Err(IoError{kind: ResourceUnavailable,
                                                        desc:"Process has no stdout", detail:None});
    let Ok(line) match BufferedStream::new(stdout).read_line();
    // more stuff...
}
````

The idea is that `match else` establishes within its scope a default set of alternative match arms, which are added to any subsequent refutable `let` that isn't comprehensive (missing its `else` or haivng a non-comprehensive set of `else` arms.)  Might be handy, but also might also be too weird.  In any case, I won't be implementing it at first.

## Next steps

Since this is pure syntactic sugar that expands to existing syntax, this should be implementable as an external syntax extension.  In this repo, I will try to do exactly that.
