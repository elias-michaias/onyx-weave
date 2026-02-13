# onyx-weave

Direct-style algebraic effects and handlers in the Onyx programming language.
This project is still super early and is therefore still a bunch of code loaded directly into main.
When it's farther along, I'll break it out into an actual library you can fetch and use.

This is a novel approach to algebraic effects implemented for curiosity's sake.
It leverages Onyx's very powerful function overload pattern matching to do the effect handler resolution purely at compiletime.
Consequently, there are no runtime checks to decide what effect to run, and Onyx's native error handling paradigm (result types) fit into this.
The function `Unwrap.suspend` take a `Result`, runs the rest of the computation if it is the vairant `.Ok`, and if not, returns a `Suspend` and short-circuits.
The `Suspend` is a resumable error that allows you to step back into the function at any point.

```fs
package main

use core {*}

#load "module.onyx"

FxError :: enum {
    PrintFoo
    GuardTriggered
}

// define a handler, statically dispatched
Console.Print.handler :: (self: Console.Print($K)) -> K {
    printf("{}.\n", self.p)
    return self.k()
}

result_that_fails :: () -> Result(str, FxError) { return .{ Err = .GuardTriggered } }

// fx! compiler extension ~= do notation in Haskell
// takes the rest of the function and wraps in callback 
// for the yielded function in last arg slot
program :: () => fx!{
    yield Console.print("init")


    x := yield Console.read()
    w := yield Console.read()
    yield Console.print(w)

    r2 := yield Unwrap.suspend(result_that_fails())
    msg := str.concat(r2, " pizza")
    yield Console.print(msg)

    return 19
}

// use interfaces to do effect row typing
console_map :: (f: () -> $T/{
    Fx.returns(i32),
    Fx.can._2(Console.t, Unwrap.t)
}) => {
    return f()
}

main :: () {
    println(typeof program)

    result := program
        |> console_map 
        |> run

    // we left the run function
    println("DONE! CONTROL FLOW IS RETURNED TO MAIN")

    // handle the error here
    // resume the error with a value and run again
    switch result {
        case .Err as err {
            err->resume("pineapple on") |> run |> println
        }
        case _ ---
    }
}
```
