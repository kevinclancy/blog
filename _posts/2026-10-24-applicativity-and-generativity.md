---
layout: post_classic
title: "6. Applicativity and Generativity"
date: 2026-10-24 09:00 -0700
categories: [understanding-module-systems]
published: true
---

Given two applications of the same functor `F`:
```
module M = F(X)
module N = F(Y)
```
should `M.t` and `N.t` be considered the same type? This post explores this question in depth and presents OCaml's answer.

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

The function `Hashtbl.create` is effectful in the sense that its result is determined not just by its arguments, but by *state* that is implicitly threaded through every expression in our OCaml code. This state influences `Hashtbl.create` to allocate distinct hash tables each time it is called, and to return references to these distinct hash tables.

So the type `symbol` depends on the value `sym2string`, which in turn depends on the effect of allocating a new hash table, which happens each time the functor is applied. Thus, a fresh opaque type `X.symbol` must be generated each time we apply `MakeSymbolTable`, as in `module X = MakeSymbolTable ()`. Thus, in the above example, `TermSymbolTable.symbol` and `TypeSymbolTable.symbol` are distinct types. A functor that generates abstract types, like `MakeSymbolTable` does, is said to be a *generative functor*. OCaml doesn't automatically infer what functors are generative by tracking effects; instead, we tell OCaml to make a functor such as `MakeSymbolTable` generative by declaring it with a unit parameter `()`.

Because some functors, like `MakeSymbolTable`, must be generative due to effects in their bodies, we could imagine a language in which all functors are generative. While that would be sound (in fact, Standard ML is one such language), it's inconvenient. In addition to generative functors like `MakeSymbolTable`, OCaml allows us to define non-generative functors. These functors are called *applicative functors*, and will be described next.

# Applicative Functors

Recall our `MakeSet` functor from the previous post. It references the following signatures `Equatable` and `Set`:
```
module type Equatable = sig
  type t
  (** The type of comparable values *)

  val eq : t -> t -> bool
  (** [eq a b] Is [a] equal to [b]? *)

  (* LAW:
     The relation Eq defined such that

     n Eq m <=> (eq n m) = true

     is an equivalence relation (reflexive, symmetric, and transitive)
  *)
end

module type Set = sig
  (**
    A signature for manipulating *sets*, which are unordered collections of unique elements
  *)

  type t
  (** An unordered collection of unique elements *)

  module Item : Equatable
  (** Defines a type [Item.t] of elements contained in sets of type [t],
      and provides [Item.t] with an equality test operator *)

  val emptyset : t
  (** [emptyset] is a set containing no elements *)

  val member : t -> Item.t -> bool
  (** [member s v] Is [v] a member of [s]? *)

  val insert : t -> Item.t -> t
  (** [insert s v] returns the smallest set containing [v] and all elements of [s] *)

  (* LAW: for all (v : Item.t),
           (member emptyset v) = false
  *)

  (* LAW: for all (v : Item.t) and (set : t),
           (member (insert set v) v) = true
  *)

  (* LAW: for all (v : Item.t) (w : Item.t) and (set : t),
           (Item.eq v w) => (member set v) = (member set w)
  *)

  (* LAW: for all (v : Item.t), (w : Item.t), and (set : t) with (Item.eq v w) = false,
           (member (insert set v) w) = (member set w)
  *)
end
```

and its implementation follows:
```
module MakeSet (Eq : Equatable) : (Set with module Item = Eq) = struct
  type t = Eq.t list

  module Item = Eq

  let emptyset = []

  let rec member (set : t) (item : Item.t) : bool =
    match set with
    | w :: rest ->
      begin match Eq.eq w item with
      | true ->
        true
      | _ ->
        member rest item
      end
    | [] ->
      false

  let insert (set : t) (item : Item.t) : t =
    match (member set item) with
    | true ->
      set
    | false ->
      item :: set
end
```

First, notice that the body of `MakeSet` produces no effects as it evaluates, because the right-hand sides of its let bindings are values: `emptyset` is bound to an empty list value, and `member` and `insert` are bound to function values. Therefore, the type `t` defined in the functor body does not depend on effects. However, it does rely on a refinement, namely that the elements of the list are equivalent to the elements inserted into the set according to `Eq.eq`:
```
type t = Eq.t list
(* Refinement: For all Eq.t lists l of type t,

    For all elements a of type Eq.t,
    there exists an element c in l such that `(Eq.eq a c) = true` if and only if some element
    b such that `(Eq.eq a b) = true` was inserted into the set earlier.
*)
```

Clearly, not all applications of `MakeSet` can share the same abstract type field `t`, because `t` has a different refinement depending on the comparison function of the argument module `Eq`. However, two applications *to the same argument* indeed yield the same refinement for `t`, so when we seal the modules resulting from the applications, the `t` type fields should be treated as identical by the type checker. OCaml takes a conservative approach to recognizing that two applications of the same functor have equal arguments: it only treats them as equal if they are *paths* which are syntactically identical. A path (written P or Q below) is either:

* A bound module identifier (like `IntEq`)
* or, a projection of a module field from a path (like `P.X`)
* or, a functor application of a path to a path (like `P(Q)`)

As an example, we have the equality `MakeSet(IntEq).t = MakeSet(IntEq).t` but not `MakeSet(struct type t = int let eq a b = (a = b) end).t = MakeSet(struct type t = int let eq a b = (a = b) end).t`.

Functors like `MakeSet`, which produce abstract type fields that are identified across multiple applications to the same arguments, are called *applicative functors*. Any functor *without* a unit parameter `()` is treated as applicative by OCaml. The following code demonstrates how applicative functors work:
```
module IntEq : (Equatable with type t = int) = struct
  (** Integers, compared by ordinary numeric equality *)
  type t = int
  let eq (a : int) (b : int) : bool = a = b
end

module TrivialIntEq : (Equatable with type t = int) = struct
  (** Collapsed integer equality where all integers are equal  *)
  type t = int
  let eq (a : int) (b : int) : bool = true
end

module SetA = MakeSet(IntEq)
module SetB = MakeSet(IntEq)
module SetC = MakeSet(TrivialIntEq)

let _ = SetA.insert SetB.emptyset 3 (* Allowed, because SetA.t = SetB.t *)
let _ = SetA.insert SetC.emptyset 3 (* Type error, because SetA.t is not equal to SetC.t *)
```

Above, the type checker considers `SetA.t` and `SetB.t` as equal, because they were obtained by applying `MakeSet` to the same path. However, `SetC` was obtained by applying `MakeSet` to a distinct path, so the type checker rejects the last line `SetA.insert SetC.emptyset 3`.

# Conclusion

Module sealing allows the module implementor to maintain precise invariants, or "refinements", on the values of abstract types. These invariants influence which abstract types can be viewed as identical from the perspective of a client using the sealed module.

When a type field's invariant depends on an effect (such as memory allocation) that is performed each time a functor is applied, each application of the functor must generate fresh abstract types. The programmer must "mark" such type-generating functors as *generative* by declaring them with a unit parameter `()`.

Even when declared type fields do not depend on effects, they may still depend on value fields of functor arguments. For example, the `MakeSet` functor produces a result whose `t` field has an invariant that depends on the argument module `Eq`'s `eq` field. Therefore, if a functor's body has no initialization effects then distinct applications to the same argument should yield identical abstract types. Such functors are called *applicative*. The programmer communicates to the OCaml compiler that a functor should be applicative by *refraining* from declaring the functor with a unit parameter `()`. The OCaml compiler takes a simplified, conservative view of when two functor arguments are equal: it only considers them equal if they are syntactically equal paths.