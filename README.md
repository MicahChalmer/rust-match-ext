# A Rust syntax extension to make `match` nicer to use

## Disclaimer

I haven't actually written the syntax extension yet.  All the expansions below were done by hand, in my head.  There are probably mistakes.  I'm putting this up in public anyway, with just this README, in order to link to it from reddit and see if I get some feedback on the design.

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

The structures being matched are more than a bunch of Results and Options, so there's no way to `try!` or `map` around it.  The comments on the pull request suggested a macro.  So here's my idea for a syntax extension that could address these problems in a general way.

## Example

I would like to create a `match!` syntax extension, such that the code below would expand into something equivalent to the pyramid snippet above:

````rust
let mut cur_struct = struct_def;
let mut structs: ~[@StructDef] = ~[];
loop {
    match! {
        let Some(t) match cur_struct.super_struct, None => break;

        let ast::TyPath(_, _, path_id) match t.node
            ^ _ => ecx.diag.handler().bug("Expected TyPath") ;

        let def_map = tcx.def_map.borrow();
        let Some(&DefStruct(def_id)) match def_map.get().find(&path_id)
            ^ _ => ecx.diag.handler().bug("Expected DefStruct");

        cur_struct = match! {
            let Some(ast_map::NodeItem(i)) match tcx.map.find(def_id.node)
                ^ _ => ecx.diag.handler().bug("Expected NodeItem");

            let ast::ItemStruct(struct_def, _) match i.node
                _ => ecx.diag.handler().bug("Expected ItemStruct");

            struct_def
        }
    };
}
````

I, for one, find that much easier to follow.  Next to each pattern, you can see the error used to escape if the pattern doesn't match.  You can read the whole thing in a sequence and understand the flow of the logic.

(Because the `bug` calls don't return, in this case the inner `match!` invocation could be eliminated, ending the whole thing with `cur_struct = struct_def` instead.  That's true of the original version as well, but there it's not so obvious--it's easy to get lost in all the nested match statements.)

## Syntax

The idea here is that inside a `match!` you can use a new type of "let" statement, which I'll call a let-match statement:

````rust
let primary_pattern match expr ^ match_arm, match_arm...
````

Where a "match arm" is the same as in a normal match statement (`pattern [ if expr] => [ expr | block  ]`).  If the primary pattern matches, the block continues with the variables from the primary pattern in scope.  Otherwise, one of the match arms that come after the `^` are used, and the value of the selected expression will be returned from the entire enclosing `match!` block, skipping what remains after the `let` statement.  In other words, this:

````rust
match! {
    // here is some stuff...
    let Foo(f) match xyz ^
        Bar => abc(),
        Baz => blargle();

    // ...and here is more stuff that uses f...
}
````

would expand into this:

````rust
{
    // here is some stuff...
    match xyz {
        Foo(f) => {
            // ...and here is more stuff that uses f...
        },
        Bar => abc(),
        Baz => blargle()
    }
}
````

Like the normal `match` statement, let-match statements must be exhaustive--the primary and secondary patterns must cover all possible values of the matched expression.  But unlike a normal `let` statement, the primary pattern can be refutable.

The benefit of this comes when match statements are nested, as shown above.  One `match!` block can replace several layers of nested `match` statements.

I used `^` in the syntax because it wasn't used elsewhere, and thought its up-arrow shape might remind you that what comes after it will be returned from the entire `match!` block rather than just the current let-match statement.  I thought of a few alternatives:
  * `let match_pat match expr else match_arm, match_arm...` - seems too verbose
  * `let match_pat match expr { match_arm, match_arm...}` - the last part looks too much like a normal match statement
I'd love to hear if there's something better.

## Interactions with other blocks

The expansion of let-match statements gets a bit weirder when they appear inside other blocks.  Consider this:

````rust
match! {
    // some stuff...
    if (x) {
        let bar = fee_fie_fo_fum();
        let Some(foo) match bar ^ None => "oh no!";
        mumble(foo);
        // more stuff inside the if statement
    } else {
        grumble("not x")
    }
    // still more stuff...
}
````

How should this expand?  We have to return out of the entire outer `match!` block from inside the inner block.  A hack with a loop can make that work:

````rust
{
    let mut res__;
    HACKLOOP: loop {
        res__ = {
            // some stuff...
            if (x) {
                match bar {
                    Some(foo) => {
                        mumble(foo);
                        // more stuff inside the if statement
                    }
                    None => { res = "oh no!"; break HACKLOOP }
                }
            } else {
                grumble("not x")
            }
            // some more stuff...
        };
        break HACKLOOP;
    }
    res__
}
````

I'm not sure if introducing a loop will have undesired effects on the optimizer, or the borrow checker, or other considerations.  For the simple if statement I can avoid introducing the loop at the cost of introducing yet another match (and wrapping the ultimate result of the expression temporarily in a `Result`...

````rust
{
    // some stuff...
    match(if (x) {
        let bar = fee_fie_fo_fum();
        match bar {
            Some(foo) => Ok({ 
                mumble(foo);
                // more stuff inside the if statement
            }),
            None => Err("oh no!");
        }
    } else {Ok({
        grumble("not x")
    })}) {
        Ok(_) => {
            // some more stuff...
        },
        Err(__esc__) => __esc
    }
}
````

...but that seems way harder to implement, because it requires much more modification of the surrounding code.  What if instead of the `if` statement the let-match occurred inside a loop?  With the loop hack, any of those scenarios would work--I'd just have to wrap the loop and assignment around the body, and then do a simple replacement of the let-match statement, with no modifications to the surrounding code.  So, loop hack it is.

## One further possible feature

The `match!` block could be used with `Result` instead of the `try!` macro.  It would have the advantage of not needing to return out of the entire fn:

````rust
match! {
    let Ok(p) match Process.new("proggy", [~"arg1", ~"arg2"]) ^ err => err;
    let Some(stdin) match p.stdin ^ None=>Err(IoError{kind: ResourceUnavailable,
                                                      desc:"Process has no stdin", detail:None});

    let Ok(_) match writeln!(stdin, "Hello there!") ^ err => err;
    let Some(stdout) match p.stdout ^ None=>Err(IoError{kind: ResourceUnavailable,
                                                        desc:"Process has no stdout", detail:None});
    let Ok(line) match BufferedStream::new(stdout).read_line()  ^ Err(IoError{kind:EndOfFile,..}) => Ok(()), err => err;
    // more stuff...
}
````

That's OK so far as it goes, but we keep repeating `^ err => err`.  Perhaps there could be a way to specify a default within each `match!` block?  It might look like:

````rust
match! {
    match else err => err;  // "match else" would declare the default non-primary match.
    let Ok(p) match Process.new("proggy", [~"arg1", ~"arg2"]);
    let Some(stdin) match p.stdin ^ None=>Err(IoError{kind: ResourceUnavailable,
                                                      desc:"Process has no stdin", detail:None});

    let Ok(_) match writeln!(stdin, "Hello there!");
    let Some(stdout) match p.stdout ^ None=>Err(IoError{kind: ResourceUnavailable,
                                                        desc:"Process has no stdout", detail:None});
    let Ok(line) match BufferedStream::new(stdout).read_line();
    // more stuff...
}
````

I'm not really sure this one is needed though--might be unnecessary complexity.  I'll probably leave it out, at least at first.

## Next steps

Since this is pure syntactic sugar that expands to existing syntax, this should be implementable as an external syntax extension.  In this repo, I will try to do exactly that.
