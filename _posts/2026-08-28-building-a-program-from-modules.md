---
layout: post_classic
title: "3. Building a Program from Modules"
date: 2026-08-28 09:00 -0700
categories: [understanding-module-systems]
published: true
---

We run a program by invoking a "main" procedure, either because we want to retrieve its outputs, or because we desire the interactive behavior or side effects that we expect it to perform. But this procedure may need to instantiate values and perform operations on them; for this, we use modules. The modules used by our main procedure may themselves be implemented in terms of other "lower level" modules.

In OCaml, it may be necessary to initialize a module before it is used. A counter module, for example, may need to allocate an integer reference storing the starting value.

We say that module `A` depends on module `B` if module `B` is referenced anywhere from within the definition of module `A`. The graph whose edges comprise the "depends on" relation is called the *module dependency graph*.  To ensure that every module is initialized before it is used, we stipulate that the module dependency graph not contain cycles. (For the most part; OCaml supports recursive modules, but they are the exception to the rule.) This is overly conservative, since a module may depend on another without needing it for initialization. However, it seems to work well enough in practice.

Roughly, we can think of modules that occur earlier in the topological sorting of the module dependency graph as lower level vocabularies and modules occurring later as higher level vocabularies. Each module adds complexity on top of the modules it depends on, both by combining multiple modules, and by combining their abstract datatypes and operations in complex ways. It compensates for the additional complexity by sealing: hiding its new datatype definitions behind abstract types of its own, and defining laws that govern the way its operations interact with those datatypes.

# The Happy Child Program

I will proceed to present an admittedly contrived and problematic example that demonstrates the above discussion using OCaml snippets.

## The "Shoes" Module

Consider this signature:
```
val tie : unit -> unit
(** Tie shoes
    Preconditions: check_tied () = false *)

val untie : unit -> unit
(** Untie shoes
    Precondition: check_tied () = true
*)

val check_tied : unit -> bool
(** Are shoes tied? *)

(* LAW:

  `check_tied () = true`
  after calling `tie ()` without any intervening calls to `untie ()`

*)

(* LAW:

  `check_tied () = false`
  after calling `untie ()` without any intervening calls to `tie ()`

*)
```

And this struct implementing the signature:
```
type shoe_state =
  | Tied
  | Untied

let shoe_state = ref Untied

let check_tied () : bool =
  match !shoe_state with
  | Tied ->
    true
  | Untied ->
    false

let tie () =
  assert (!shoe_state = Untied);
  shoe_state := Tied

let untie () =
  assert (!shoe_state = Tied);
  shoe_state := Untied
```

OCaml initializes modules by normalizing each of the expressions on the right-hand side of its let bindings to values. It processes the let bindings from top to bottom. `shoe_state` is bound to the expression `ref Untied`, which executes by allocating a reference cell and storing the value `Untied` in the reference cell. However, `check_tied`, `tie`, and `untie` are function abstractions, so they are already in normal form.

Unlike the modules defined in previous posts, this module's `tie` and `untie` operations perform side effects. Since they only perform effects on a single piece of global state (the allocated reference cell), it would be extremely easy for two different modules to interfere with each other's usage of `Shoes`, so creating modules like this is ill-advised; the remedy for this will be presented later.

## The Happy Child Module, Take 1

From the lower level `Shoes` module, we construct a higher level `HappyChild` module.
```
val show_off : unit -> unit
(** Brag about skills via standard output *)
```

Here is its implementation
```
let show_off () =
  (match Shoes.check_tied () with
  | false ->
    Printf.printf "My shoes are untied!\n";
    Shoes.tie ();
    Printf.printf "Not any more!\n"
  | true ->
    Printf.printf "My shoes are tied!\n");

  Shoes.untie ();
  Printf.printf "Now they're untied!\n"
```

## The Counter Module

Children also like to count, so let's make a `NumCounter` module. Here is its signature:
```
type t
(** A counter that counts natural numbers upward *)

val create : unit -> t
(** Create a new counter starting from 0 *)

val get_next : t -> int
(** Get the next natural number *)

(* LAW:
   get_next (create ()) = 0
*)

(* LAW:
  Each call to `get_next x` after the first produces the natural number directly after the previous call to `get_next x`.

  In pseudocode:
   for all x:t.
     let a = get_next x in
     let b = get_next x in
     assert (b = a + 1)
*)
```

Here is a struct implementing this signature:
```
type t = int ref

let create () : t =
  ref 0

let get_next (counter : t) : int =
  let ret = !counter in
  counter := !counter + 1;
  ret
```

Unlike `Shoes`, this struct's fields consist solely of function abstractions, so no initialization is performed. Its operations do perform mutable state updates, but the state is mediated by values of type `NumCounter.t`: reference cells hidden behind an abstract type. These values are created by the calls to the `create` operation. Even though `NumCounter` is globally accessible, each client can keep its own value of type `NumCounter.t` private so that other modules can't interfere with its state.

## The Happy Child Module, Take 2

Here is an updated version of the `HappyChild` implementation that uses `NumCounter`:
```
let show_off () =
  (match Shoes.check_tied () with
   | false ->
     Printf.printf "My shoes are untied!\n";
     Shoes.tie ();
     Printf.printf "Not any more!\n"
   | true ->
     Printf.printf "My shoes are tied!\n");

  Shoes.untie ();
  Printf.printf "Now they're untied!\n";
  Printf.printf "Now I will count from 0 to 2!\n";

  let counter = NumCounter.create () in
  Printf.fprintf stdout "%d\n" (NumCounter.get_next counter);
  Printf.fprintf stdout "%d\n" (NumCounter.get_next counter);
  Printf.fprintf stdout "%d\n" (NumCounter.get_next counter)
```

To actually execute the `show_off` function, OCaml's build system Dune requires us to identify a "main" module whose initialization is synonymous with running the program. Let's call ours `Main`. We refrain from sealing it with a signature, defining it only using a file `main.ml` which contains a single let binding:
```
let () =
  HappyChild.show_off ()
```

OCaml's build system Dune culls out any modules that do not transitively depend on `Main` and then topologically sorts them into an initialization order. It starts with a module dependency DAG which looks like this:
<figure>
<img
  src="/assets/images/building-a-program-from-modules/module-dag.png"
  style="margin-top: 30px; margin-bottom: 30px"
>
</figure>

Since neither `NumCounter` nor `Shoes` depends on the other, there are two orders that Dune could topologically sort the modules into. It resolves the ordering alphabetically, placing `NumCounter` before `Shoes`, giving the following topologically sorted module list:
```
NumCounter
Shoes
HappyChild
Main
```
Execution then initializes each module from top to bottom, where modules are initialized by normalizing the right-hand-sides of each of its let bindings from top to bottom. By the time `Main` is initialized, `HappyChild` has already been initialized (and transitively both `NumCounter` and `Shoes`) so that we can invoke its `show_off` operation.

I'm not sure if I like that initialization of independent modules is ordered alphabetically. That seems chaotic. It may be better to require the programmer to provide an initialization list themselves. But that's not how Dune handles things.

## The Alphabet Iterator

You may be getting tired of this contrived example. But we're almost done. I will only add *one* more module, called `AlphabetIter`, which is sort of like `NumCounter` but instead iterates through letters of the alphabet.
```
(** Iterates through the upper case alphabet starting from 'A' *)

val get_next : unit -> char option
(** Gets the next alphabetic character, or None if all characters have been exhausted *)
```

Here is its implementation:
```
let counter = NumCounter.create ()

let get_next () : char option =
  let ret = (NumCounter.get_next counter) + (Char.code 'A') in
  match (ret <= (Char.code 'Z')) with
  | true ->
    Some(Char.chr ret)
  | false ->
    None
```

`AlphabetIter` initializes itself by creating a `NumCounter.t` value called `counter`. This is used to offset the ascii codes for alphabetic characters starting at `A`.

Unlike the `NumCounter` module, which serves as a "factory" that has no initialized mutable state, but allocates new pieces of mutable state each time `create` is called, the `AlphabetIter` module serves as an "object", allocating a single piece of mutable state during its initialization, similar to the `Shoes` module. As explained in the `Shoes` section, making top-level modules object-style modules that own mutable state is often a bad idea, as its various client modules have no effective way of coordinating their state updates.

We update `HappyChild` to use `AlphabetIter`:
```
let show_off () =
  (match Shoes.check_tied () with
   | false ->
     Printf.printf "My shoes are untied!\n";
     Shoes.tie ();
     Printf.printf "Not any more!\n"
   | true ->
     Printf.printf "My shoes are tied!\n");

  Shoes.untie ();
  Printf.printf "Now they're untied!\n";
  Printf.printf "Now I will count from 0 to 2!\n";

  let counter = NumCounter.create () in
  Printf.fprintf stdout "%d\n" (NumCounter.get_next counter);
  Printf.fprintf stdout "%d\n" (NumCounter.get_next counter);
  Printf.fprintf stdout "%d\n" (NumCounter.get_next counter);

  Printf.fprintf stdout "I know the alphabet!\n";
  Printf.fprintf stdout "%c\n" (Option.get (AlphabetIter.get_next ()));
  Printf.fprintf stdout "%c\n" (Option.get (AlphabetIter.get_next ()));
  Printf.fprintf stdout "%c\n" (Option.get (AlphabetIter.get_next ()))
```

The reason `AlphabetIter` has been included in this post is because of the way it affects the module dependency graph:
<figure>
<img
  src="/assets/images/building-a-program-from-modules/module-dag-alphabet.png"
  style="margin-top: 30px; margin-bottom: 30px"
>
</figure>

This shows that a module dependency graph need not be a tree, but can be any directed acyclic graph. Recall this paragraph from the introduction:
```
Roughly, we can think of modules that occur earlier in the topological sorting of the module dependency graph as lower level vocabularies and modules occurring later as higher level vocabularies implemented in terms of earlier modules as higher level vocabularies. Each module adds complexity on top of the modules it depends on, both by combining multiple modules, and by combining their abstract datatypes and operations in complex ways. It compensates for the additional complexity by sealing: hiding its new datatype definitions behind abstract types of its own, and defining laws that govern the way its operations interact with those datatypes.
```

The graph above shows us that the "lower level vocabularies" (`AlphabetIter` and `NumCounter`) that "higher level vocabularies" (`HappyChild`) are defined from need not be independent. In this case, one of `HappyChild`'s dependencies `AlphabetIter` was defined from another of its dependencies `NumCounter`.

The initialization order for the above modules is the topological sorting Dune produces from the dependency graph:
```
NumCounter
AlphabetIter
Shoes
HappyChild
Main
```

An important final point is that after topologically ordering our top-level modules, we can combine them into a single abstract syntax tree by listing their definitions from top to bottom:
```
module type NumCounter = sig
  ...
end

module NumCounter : NumCounter = struct
 ...
end

module type AlphabetIter = sig
 ...
end

module AlphabetIter : AlphabetIter = struct
 ...
end

...

module Main = struct
  ...
end
```
The operational semantics of our full project then amounts to initializing the modules in the above syntax tree from top-to-bottom. In this way, we can view a multi-file OCaml project as a single syntactic object amenable to formal reasoning.

# Conclusion

In this post, I've shown--through an admittedly contrived and questionably engineered example--how a software project can be viewed as a "vending machine" (module) implemented in terms of other "lower level vending machines" (also modules).

OCaml formulates the process of running a program as initializing a sequence of top-level modules. While theoretically elegant, I retain some skepticism of this idea, because allowing modules to perform side effects in an ordering that is not explicitly visible to the programmer is inherently chaotic.