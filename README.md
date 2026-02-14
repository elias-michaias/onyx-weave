# onyx-weave

Direct-style algebraic effects and handlers in the Onyx programming language.
This project is still super early and is therefore still a bunch of code loaded directly into main.
When it's farther along, I'll break it out into an actual library you can fetch and use.

This is a novel approach to algebraic effects implemented for curiosity's sake.
It leverages Onyx's very powerful function overload pattern matching to do the effect handler resolution purely at compiletime.
Consequently, there are no runtime checks to decide what effect to run, and Onyx's native error handling paradigm (result types) fit into this.
The function `Unwrap.suspend` takes a `Result`, runs the rest of the computation if it is the variant `.Ok`, and if not, returns a `Suspend` and short-circuits.
The `Suspend` is a resumable error that allows you to step back into the function at any point.

```fs
package main

use core {*}

#load "module.onyx"

FxError :: enum {
    PrintFoo
    GuardTriggered
}

res_fail :: () -> Result(str, FxError) { return .{ Err = .GuardTriggered } }
res_succeed :: () -> Result(str, FxError) { return .{ Ok = "bruuhhhh" } }
opt_fail :: () -> Optional(str) { return .{} }
opt_succeed :: () -> Optional(str) { return .{ Some = "swag" } }

program :: () => fx!{
    yield Console.print("init")


    x := yield Console.read()
    w := yield Console.read()
    yield Console.print(w)

    r2 := yield Unwrap.suspend(res_fail())
    msg := str.concat(r2, " pizza")
    yield Console.print(msg)

    return 19
}

console_map :: (f: () -> $T/{
    Fx.returns(i32),
    Fx.can._2(Console.t, Unwrap.t)
}) => {
    return f()
}


main :: () {
    println(typeof program)

    handle :: #match {
        (self: Console.Print($K)) => {
            printf("{}!\n", self.p)
            return self.k()
        }
        (self: $T/FxHandleable) => self->handle()
    }

    result := () => {
        handle :: #match {
            (self: Console.Print($K)) => {
                printf("{}?\n", self.p)
                return self.k()
            }
            (self: $T/FxHandleable) => self->handle()
        }

        return program
            |> console_map 
            |> run
    } 

    println("DONE! CONTROL FLOW IS RETURNED TO MAIN")

    switch result() {
        case .Err as err {
            err->resume("pineapple on") |> run |> println
        }
        case _ ---
    }
}












```

Outputs:
```
() -> Console.Print(Console.Read(Console.Read(Console.Print(Unwrap.Suspend(FxError, [] u8, Console.Print(i32))))))
DONE! CONTROL FLOW IS RETURNED TO MAIN
init?
foo
bar
bar?
pineapple on pizza!
19
```
