# onyx-weave

**Algebraic effects and handlers for the Onyx programming language — zero heap allocation, compile-time dispatch, direct style.**

> [!NOTE]
> This library is currently a proof-of-concept. Everything is subject to change.

---

## What this is

`onyx-weave` is an algebraic effect system with an unusual set of properties that make it viable for systems programming:

- **Effect row polymorphism** — Use interfaces like `Fx.can._2(Console.t, Unwrap.t)` to allow effect permissions for functions and reject non-conforming programs at compile-time.
- **Regular language over effects** — Use interfaces like `Fx.seq.cant._2(Fs.Read.t, Fs.Write.t)` to explicitly disallow an exact sequence of effects within an effectful computation.
- **Arbitrary effect type composition** — Effect types like `Console.t` utilize an overloaded `handle` function which can fit any type. Users can easily create custom effect types that include any arbitrary set of effects, allowing for granular and coarse effect permissions arbitrarily.
- **Zero heap allocation by default** — including for non-determinism. Effects use explicit context structs passed by value. The continuation is a plain function pointer and the captured state is a stack copy of only the live variables at each effect boundary.
- **Compile-time handler dispatch** — no runtime type tags, no vtable, no dynamic dispatch. Handler resolution is purely a function of the types involved, resolved by Onyx's overload system before the program runs.
- **Direct style** — you write imperative-looking code. The `fx!{}` macro rewrites it to continuation-passing style automatically, threading context structs through the generated continuations. The `fx!{}` performs symbol analysis so that it only copies the symbols into context that you need, preventing the overhead of traditional stack copying.
- **Statically enforceable memory budgets** — because context struct size is always known at compile time, you can enforce hard upper bounds on effect memory usage as a compile error via `Fx.max_ctx_size(N)`.
- **Modular handlers** — handlers are types with namespaced overload sets. You can define effects and handlers in separate modules and compose them at the call site. The type system verifies compatibility at compile time.

The effect chain is encoded in the return type of every effectful function. A function that prints, then reads, then prints again has a return type that structurally reflects that sequence. This means the compiler knows the full effect signature of every function without any annotations — it falls out of the types automatically.

---

## How it works

### Effects as types

Each effect is a concrete struct carrying a payload and a continuation:

```onyx
Console.Print :: struct ($K: type_expr, $C: type_expr) {
    p:   str      // payload — what we're asking the handler to do
    k:   ($C) -> $K  // continuation — the rest of the program
    ctx: $C       // captured context — live variables at this effect boundary
}
```

The handler receives the struct, inspects the payload, and calls `k` with the context to resume. No heap allocation. No closure environment. The context struct contains exactly the variables that are live at this point in the computation — determined statically by the `fx!` rewriter.

### Direct style via `fx!`

You write this:

```onyx
program :: () => fx!{
    x := perform Each.choose(.["yes", "no"])
    msg := str.concat("cool: ", x)
    perform Console.print(msg)
    y := perform Console.read()
    perform Console.print(y)
    return 12
}
```

The `fx!` macro rewrites it to CPS, inserting context structs and continuation functions:

```onyx
return Each.choose(.["yes", "no"], 0, (__ctx__, x) => {
    msg := str.concat("cool: ", x)
    return Console.print(msg, .{x=x, msg=msg}, (__ctx__) => {
        return Console.read(.{x=__ctx__.x}, (__ctx__, y) => {
            return Console.print(__ctx__.y, 0, (__ctx__) => {
                return 12
            })
        })
    })
})
```

Only the variables that are actually needed downstream get captured. `msg` is captured at the `Console.print` boundary because it's used there, then dropped. `x` is threaded further because it's needed by `Each.guard` later. The rewriter computes this statically.

### Handlers as namespaced types

A handler is a type with an overloaded `handle` function. You pass the type — not a value — to `run`:

```onyx
MyHandler :: struct {
    handler :: #match {
        macro (e: Console.Print($K, $C)) => {
            printf("{}\n", e.p)
            return e.k(e.ctx)
        }
        macro (e: Console.Read($K, $C)) => {
            line := stdio |> Reader.make |> .read_line |> .strip_whitespace
            return e.k(e.ctx, line)
        }
        macro (e: $T/FxHandleable) => e->handle()
    }
}

result := program() |> run(MyHandler)
```

Because `MyHandler` is a type, its entire namespace is available to the `run` macro. This means handlers can carry arbitrary supporting definitions — allocators, loggers, configuration — that effects can call back into without any additional plumbing. The handler type is a capability bundle.

### Effect row constraints

Functions can constrain the effects their arguments perform:

```onyx
console_map :: (f: () -> $T/{
    Fx.returns(i32),
    Fx.can._2(Console.t, Unwrap.t),
    Fx.max_ctx_size(16)
}) => {
    return f()
}
```

This says: `f` must return `i32`, may only perform `Console` and `Unwrap` effects, and no context struct in its effect chain may exceed 16 bytes. The compiler verifies all three at the call site. `Fx.seq.can._2` additionally enforces ordering — `Console` effects must appear before `Unwrap` effects.

These constraints are checked by crawling the type-level effect chain at compile time. There is no runtime representation of the constraint.

---

## Performance

| | Cost |
|---|---|
| Handler dispatch | Zero — resolved at compile time via overload matching |
| No-capture effects | One indirect function call |
| Capturing effects | One indirect function call + `sizeof(ctx)` bytes copied |
| Multi-shot (non-determinism) | Same as capturing — context is immutable, no copies needed |
| Heap allocation | None by default |

The cost model is flat: one indirect call plus the size of your live state at each effect boundary. There are no runtime type checks, no vtables, no allocator involvement. Any allocation in your program is from your data, not from the abstraction.

With a sufficiently mature compiler that can devirtualize the function pointer call (which is possible since the call target is statically known), no-capture effects approach the cost of a direct function call.

---

## Example

```onyx
package main

use core {*}
use core.io { Reader }
use weave {*}

#load "src/module.onyx"

FxError :: enum {
    GuardTriggered
}

res_fail    :: () -> Result(str, FxError) { return .{ Err = .GuardTriggered } }
res_succeed :: () -> Result(str, FxError) { return .{ Ok = "mushroom on" } }

program :: () => fx!{
    perform Console.print("init")

    w := perform Console.read()
    perform Console.print(w)

    r2 := perform Unwrap.suspend(res_fail())
    msg := str.concat(r2, " pizza")
    perform Console.print(msg)

    return 19
}

fx_apply :: (f: () -> $T/{
    Fx.returns(i32),
    Fx.can._2(Console.t, Unwrap.t),
    Fx.max_ctx_size(16)
}) => {
    return f()
}

main :: () {
    println(typeof program)

    result := program
            |> fx_apply
            |> run(struct {
                handler :: #match {
                    (self: Console.Print($K, $C)) => {
                        printf("{}???\n", self.p)
                        return self.k(self.ctx)
                    }
                    (self: Console.Read($K, $C)) => self->handle()
                    (e: Unwrap.Suspend($E, $T, $K/FxHandleable, $C)) -> Unwrap.Suspend(E, T, typeof K.{} |> handler |> run, C) {
                        switch e {
                            case .Ok as ok do return .{ Ok = handler(ok) |> run }
                            case .Err as err do return .{ Err = err }
                        }
                    }
                }
            })

    println("CONTROL FLOW IS RETURNED TO MAIN.")

    handler :: #match {
        (self: Console.Print($K, $C)) => {
            printf("{}!!!\n", self.p)
            return self.k(self.ctx)
        }
    }

    log := switch result {
        case .Err as err => err
            |> .resume("pineapple on") 
            |> run 
        case .Ok as ok => ok
    }

    println(log)
}
```

Output:

```
() -> Console.Print(Console.Read(Console.Print(Unwrap.Suspend(FxError, [] u8, Console.Print(i32, i32), i32), i32), i32), i32)
init???
spaghetti
spaghetti???
CONTROL FLOW IS RETURNED TO MAIN.
pineapple on pizza!!!
19
```

The first line is the inferred type of `program` — the full effect chain, printed by the compiler. Every effect the function performs is visible in its return type without any annotations.

---

## Why Onyx

This design requires a specific combination of language features that is difficult to replicate elsewhere:

- **Types as namespaces** — handler types carry their entire overload set as named definitions, accessible to the `run` macro at expansion time
- **Structural overload dispatch** — `#match` resolves to different handler implementations based on the concrete effect type, without any runtime tag
- **Compile-time type destructuring** — `Console.Print($K, $C)` extracts the type parameters of a concrete generic type in a function head
- **Macros with guaranteed inlining** — the `macro` keyword ensures handler dispatch is expanded at the call site, not called through an additional indirection
- **Type-level state machines** — the `Fx.seq` constraints use phantom boolean parameters to enforce ordering of effects in the type checker

Most languages have one or two of these. Onyx has all of them, and the combination is what makes zero-overhead modular effects possible. In particular: you cannot pass an entire overload set as a first-class argument in C++, Rust, or Go. This is the property that makes handler modularity and zero-cost dispatch mutually achievable rather than a tradeoff.
