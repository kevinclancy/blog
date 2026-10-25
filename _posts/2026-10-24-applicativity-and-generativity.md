---
layout: post_classic
title: "6. Applicativity and Generativity"
date: 2026-10-24 09:00 -0700
categories: [understanding-module-systems]
published: false
---


# Invariants

Consider the following signature and functor:
```
module type SymbolTable = sig
  type symbol
  (** A symbol which occurs one or more times in a program *)

  val get_symbol : string -> symbol
  (** [get_symbol str] Looks up a symbol named [str],
      or generates and returns a new one if no such symbol exists *)

  val get_name : symbol -> string
  (** [get_name sym] Returns the string name of symbol [sym] *)
end

module MakeSymbolTable () : SymbolTable = struct
  type symbol = int

  let string2sym : (string, symbol) Hashtbl.t = Hashtbl.create 64
  let sym2string : (symbol, string) Hashtbl.t = Hashtbl.create 64
  let next_sym : int ref = ref 0

  let get_symbol (name : string) : symbol =
    match Hashtbl.find_opt string2sym name with
    | Some sym ->
      sym
    | None ->
      let sym = !next_sym in
      next_sym := !next_sym + 1;
      Hashtbl.add string2sym name sym;
      Hashtbl.add sym2string sym name;
      sym

  let get_name (sym : symbol) : string =
    Hashtbl.find sym2string sym
end
```

A `SymbolTable` is used by a compiler. It associates string identifiers read in by a parser with *symbols* for comparatively efficient equality testing and storage. Since languages generally have multiple namespaces (for example, OCaml has separate namespaces for terms, types, and modules), we want the ability to instantiate multiple modules which all implement the `SymbolTable` interface. For this, we have created the functor `MakeSymbolTable`, which produces a different symbol table at each application:
```
module TermSymbolTable = MakeSymbolTable ()
module TypeSymbolTable = MakeSymbolTable ()
module ModuleSymbolTable = MakeSymbolTable ()
```

An important aspect of the `MakeSymbolTable` definition is that the `get_name` function is total: regardless of what argument is passed to it, it never throws an exception. But it's implemented in terms of `Hashtbl.find`, which *does* throw an exception when its key argument `sym` is not contained as a key in its table argument `sym2string`. How can this be?

`get_name` is total despite being implemented in terms of `Hashtbl.find` because of an *invariant* the struct ensures: all values of type `symbol` are integers that occur as keys in `sym2string`. The invariant can be proven by examining the definition of the struct produced by `MakeSymbolTable`. First, we make note of another invariant, which is easily proven as an exercise: any symbol value stored in `string2sym` is stored as a key in `sym2string`. Then, we note that the struct only emits `symbol` values via its `get_symbol` method, but in both branches of this method, the emitted symbol is a key in `sym2string`. In the first branch, the emitted symbol was already stored as a value in `string2sym` before `get_symbol` was called, and hence it was already stored as a key in `sym2string`. In the second branch, the returned symbol is added as key by the call `Hashtbl.add sym2string sym name` immediately before it is returned.

# Generative Functors

Now, we choose to view this invariant as an informal refinement on the `int` type used as the definition of `symbol`:
```
module MakeSymbolTable () : SymbolTable = struct
  type symbol = int (* Refinement: symbol is the type of integers x such that x occurs as a key in sym2string *)
  ...
end
```
This *refinement* is an "informal subtype" of `int` that is too precise for OCaml's type system to track. However, as the implementor of the body of `MakeSymbolTable`, we were able to verify in the discussion above that the refinement is indeed valid. Stepping back, this demonstrates an important aspect of the power of module systems. Because the refinement refers to `sym2string`, it doesn't make sense in contexts outside of the functor body, for which `sym2string` is not in scope. Contexts outside the functor body are nonetheless still able to refer to this type *abstractly* as `symbol`.

Now, consider the following code:
```
module TermSymbolTable = MakeSymbolTable ()
module TypeSymbolTable = MakeSymbolTable ()
let () =
  let sym_a = TermSymbolTable.get_symbol "a" in
  let str_a = TypeSymbolTable.get_name sym_a in
  Printf.printf "%s" str_a
```

This code, if it were well typed, would raise a `Not_found` exception during the call `TypeSymbolTable.get_name sym_a`. The problem is that `sym_a` was produced by the `TermSymbolTable` module, which guarantees that all symbols are keys of the `sym2string` hash table *contained in* `TermSymbolTable`. However, the `TypeSymbolTable` module contains a *distinct* `sym2string` hash table, which is empty at the time of the call to `TypeSymbolTable.get_name sym_a`.

More precisely, while the refined view of the concrete representations underlying `TermSymbolTable.symbol` and `TypeSymbolTable.symbol` may both look like this:
```
type symbol = int (* Refinement: symbol is the type of integers x such that x occurs as a key in sym2string *)
```

They are in fact distinct representations, because `sym2string` refers to distinct hash tables in each representation. The issue is that `sym2string` is defined in the body of `MakeSymbolTable` using an *effectful* expression:
```
let sym2string : (symbol, string) Hashtbl.t = Hashtbl.create 64
```

The function `Hashtbl.create` is effectful in the sense that its result is determined not just by its arguments, but by *state* that is implicitly threaded through every expression in our OCaml code. This state influences `Hashtbl.create` to allocate distinct hash tables each time it is called, and returns references to these distinct hash tables.

