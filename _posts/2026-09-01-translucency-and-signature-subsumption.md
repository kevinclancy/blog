---
layout: post_classic
title: "4. Translucency, Signature Subsumption, and Sharing Constraints"
date: 2026-09-01 09:00 -0700
categories: [understanding-module-systems]
published: true
---

# Translucency

Previously, I defined signatures as a declarations of three types of components:
* Data types defining the sorts of entities that can be manipulated
* Operations to be performed on the entities
* Laws constraining what behaviors the signature's clients may observe when invoking a sequence of its operations

But I presented a limited view of the first type of component: datatypes. They were presented as opaque names, signifying the existence of datatypes without defining their representations: in other words, abstract types. An example of such an abstract type component is the `t` component of the `AgeMap` example from the first post:
```
module type AgeMap = sig
  type t
  val insert : t -> string -> int -> unit
  val get : t -> string -> int
  ...
end
```

However, in addition to these *opaque* type declarations, OCaml signatures may also contain *transparent* type declarations, such as `name` below:
```
module type AgeMap = sig
  type t
  (** A map from names to ages *)

  type name = string
  (** The type of names used by the age map *)

  val insert : t -> name -> int -> unit
  val get : t -> name -> int

  ...
end
```

A transparent type declaration like `name` defines exactly how the datatype is represented. In this case, the transparent declaration indicates to clients of the `AgeMap` signature that any struct implementing AgeMap implements `name` using `string`. Clients can therefore use values of type `string` where values of type `AgeMap.name` are expected, e.g. given a value `v` of type `AgeMap.t` a client of a module `M` implementing the `AgeMap` signature can perform the function application `M.insert age_map "Ralph" 31`.

The combination of opaque and transparent type declarations is often called *translucency*. At first, it may seem superficial; above, `name` simply serves as an alias for the type `string`. But at a practical level, the aliases may name complex datatypes that become unwieldy to type out multiple times across a signature. Furthermore, the right-hand side of a transparent type may be a complex type expression whose component types include both abstract types declared within the same signature, and concrete types visible from outside the signature, for example:
```
module type KVMap = sig
  type t
  type key = string
  type value
  type entry = key * value
  ...
end
```

There are yet more fundamental reasons to include transparent type declarations, which I will proceed to discuss.

# Signature Subsumption

Suppose we have a module M sealed with signature S. If T is a signature stipulating "fewer requirements" than S, we should be able to pass M into contexts where modules of signature T are expected. When this is the case, we say that S is subsumed by T and write S <: T.

As a first approximation, a signature imposes various types of requirements on the structs considered to implement it:

1. For an opaque declaration of a type named t, the struct is required to have a type component named t.
2. For a transparent declaration of a type named t as the type $$\tau$$, the struct must have a type component named t defined as $$\tau$$.
3. For a value declaration "val x : $$\tau$$", the struct must have a value binding named x whose type is inferred or ascribed as $$\tau$$.
4. And *informally*, the value declarations are required to satisfy all laws included in the signature.

However, this is an oversimplification. It overlooks the dependencies between type declarations and value fields. For example, consider checking that the below struct `FooImpl` implements the signature `Foo`:
```
module type Foo = sig
  type t
  type s = t * int
  val bar : s -> unit
end

module FooImpl : Foo = struct
  type t = int
  type s = int * int
  let bar (arg1 : int * t) = ()
end
```

First, to check that $$\mathtt{FooImpl}$$ satisfies the requirement of possessing $$\mathtt{type~t}$$ declared in $$\mathtt{Foo}$$, we note that $$\mathtt{FooImpl}$$ does indeed contain the definition of a type field named $$\mathtt{t}$$: $$\mathtt{type~t} = \mathtt{int}$$. Next, we check that the transparent type declaration $$\mathtt{type~s} = \mathtt{t} * \mathtt{int}$$ is satisfied by the $$\mathtt{FooImpl}$$ struct. However, $$\mathtt{FooImpl}$$ has $$\mathtt{type~s} = \mathtt{int} * \mathtt{int}$$, which does not match $$\mathtt{Foo}$$'s $$\mathtt{type~s} = \mathtt{t} * \mathtt{int}$$ verbatim; instead, $$\mathtt{int} * \mathtt{int}$$ is *equivalent* to $$\mathtt{t} * \mathtt{int}$$ under a *type equation context*, established by checking the $$\mathtt{type~t}$$ declaration, containing the sole equation $$\mathtt{t} = \mathtt{int}$$. Likewise, we must check that the $$\mathtt{bar}$$ field of $$\mathtt{FooImpl}$$'s inferred type $$\mathtt{int} * \mathtt{t} \to \mathtt{unit}$$ is equivalent to $$\mathtt{s} \to \mathtt{unit}$$ under a context consisting of the two equations $$\mathtt{t} = \mathtt{int}$$ and $$\mathtt{s} = \mathtt{int} * \mathtt{int}$$.

This suggests an algorithm for checking that S <: T is true.

1. For each opaque declaration "type t" in T, check that S declares a type named t which is either opaque ("type t") or transparently defined as any type $$\tau$$ ($$\mathtt{type~t} = \tau$$); in the latter case, insert the equation $$\mathtt{t} = \tau$$ into the context before processing subsequent declarations.
2. For each transparent declaration $$\mathtt{type~t} = \tau$$ in T, check that S declares $$\mathtt{type~t} = \tau'$$, where $$\tau'$$ is equivalent to $$\tau$$ under the current context. Insert $$\mathtt{t} = \tau$$ into the context before processing subsequent declarations.
3. For each declaration $$\mathtt{val~x} : \tau$$ in T, check that S has a declaration $$\mathtt{val~x} : \tau'$$, where $$\tau$$ is equivalent to $$\tau'$$ under the current context.
4. Check that the conjunction of the laws of S imply the conjunction of the laws of T.

Points (1)-(3) are handled by the OCaml type checker, but point (4) is beyond its scope. Also note that when S <: T, S may contain type and value components whose names do not match the names of any type and value components of T.

As a final note, signature subsumption checking should be robust to the order that independent type declarations are listed in. In particular, the following signatures should mutually subsume each other:
```
module type S = sig
  type t
  type s
end

module type T = sig
  type s
  type t
end
```

All this is to say that checking signature subsumption is rather complex and technical. The *intuition* behind it, however, is simple: S <: T means "T imposes a weaker requirement on the structs that implement it than S does" or, equivalently, "any struct that implements S also implements T".

# Sharing Constraints

So far, we've seen one signature constructor form:

$$\mathsf{sig~...~end}$$

Above, $$\mathsf{...}$$ is a placeholder for a list of the signature's components.

The *sharing constraint* is another useful signature constructor. It has the following form:

$$\mathsf{S~with~type~t} = \tau$$

Above, $$\mathsf S$$ must be a signature that has an opaque type component named t and $$\tau$$ may be an arbitrary type expression. This constructor is considered ill formed whenever either $$\mathsf S$$ has no type component t or S has a transparent type component t. Thus, whenever the signature $$\mathsf{S~with~type~t} = \tau$$ is well formed, we know that $$(\mathsf{S~with~type~t} = \tau) <: \mathsf S$$.

Here is an example usage of sharing constraints. Recall the `TraverseGraph` signature from [my first post]({% post_url 2026-08-23-introduction %}):
```
module type TraverseGraph = sig
  (** A directed graph.
      This graph is static, in that traversability and adjacency do not change over time.
  *)

  type node
  (** A node in a graph *)

  val compare_node : node -> node -> int
  (** [compare_node n m] Returns 0 iff node [n] equals node [m] *)

  val adjacent_nodes : node -> node list
  (** [adjacent_nodes n] All traversable nodes adjacent to node [n] *)

  (* LAW:
     The relation Eq defined such that

     n Eq m <=> (compare_node n m = 0)

     is an equivalence relation (reflexive, symmetric, and transitive)
  *)

  (* LAW:
     The relation Lt defined such that

     n Lt m <=> (compare_node n m < 0)

     is a strict total order (transitive, irreflexive, and trichotomy w.r.t. Eq)
  *)

  (* LAW:
    n Eq m => (adjacent_nodes n = adjacent_nodes m)
  *)
end
```

Consider the following graph:
<figure>
<img
  src="/assets/images/translucency-and-signature-subsumption/example-graph.png"
  style="margin-top: 30px; margin-bottom: 30px"
>
</figure>

A struct implementing `TraverseGraph`, using `int` as the `node` type and providing the above graph follows:
```
module MyGraph : (TraverseGraph with type node = int) = struct
  type node = int
  let compare_node (a : int) (b : int) = a - b
  let adjacent_nodes (n : int) =
    match n with
    | 0 -> [1 ; 2]
    | 1 -> [0 ; 2 ; 3]
    | 2 -> [0 ; 1]
    | 3 -> [1]
    | _ ->
      failwith "invalid node"
end
```

Then we can pass the struct into our BFS function and print its result:
```
let () =
  let opt_path = shortest_path (module MyGraph) 0 2 in
  match opt_path with
  | Some(path) ->
    List.iter (fun n -> Printf.printf "%d\n" n) path
  | None ->
    Printf.printf "no path\n"
```

which prints the following output:
```
0
2
```
Sealing `MyGraph` with $$\mathsf{TraverseGraph}~\mathsf{with}~\mathsf{type}~\mathsf{node} = \mathsf{int}$$ ensures that if they type checker succeeds then `MyGraph` *can* be sealed with $$\mathsf{TraverseGraph}$$ in the future without *initially* sealing off `node` with an abstract type.

If we had sealed `MyGraph` using `TraverseGraph` instead of the sharing constrained version, we would not have been able to call `shortest_path`, because there would have been no way to obtain any elements of type `MyGraph.node`. In fact, sealing directly with `TraverseGraph` would leave us with a totally useless module: with no way to obtain an element of type `MyGraph.node`, neither of the operations `adjacenct_nodes` nor `compare_node` could be called.

This example has demonstrated just *one* way to use a type sharing constraint. There are multiple others that we will see shortly in the following posts.

## Different Kinds of Opaque Declarations

Recall the `Queue` signature from [the first post]({% post_url 2026-08-23-introduction %}), which I'll reproduce below:
```
module type Queue = sig
  (** The standard functional queue ADT *)

  type 'a t
  (** A functional queue whose elements have type ['a] *)

  val empty : 'a t
  (** An empty queue *)

  val push : 'a t -> 'a -> 'a t
  (**
    [q' = push q v] pushes element [v] onto the back of queue [q], yielding [q']
  *)

  val pop : 'a t -> 'a * 'a t
  (** [(a, q') = pop q] Pops front [a] of the queue [q], yielding [q']

      ## Requires

      * [not (is_empty q)]

      ## Returns

      * a:'a - The element popped from the front of [q]

      * q':'a t - The queue resulting from popping the front off [q]
  *)

  val is_empty : 'a t -> bool
  (** [is_empty q] Is queue [q] empty? *)

  (* LAW:
    The sequence underlying the queue named "empty" (defined above) is the empty sequence
  *)

  (* LAW:

    If s is the sequence of elements of type 'a underlying a queue q then
    [is_empty q] is true iff s is the empty sequence

  *)

  (* LAW:

     If s is the sequence of elements underlying a queue q,
     then the sequence underlying [push q a] is s ++ [a] (a appended to the end of s)

  *)

  (* LAW:

     If [a] ++ s (element a prepended to the sequence s) is the sequence of elements underlying
     a queue q, then [pop q] = (a, q'), where the sequence underlying queue q' is s
  *)
end
```

It declares an opaque family of types `'a t`. These opaque types are fundamentally different than the opaque type `TraverseGraph.node`, in the sense that they are intended to hide their concrete representations. These type declarations occur in signatures that seal structs immediately when the structs are created. On the other hand, the `TraverseGraph.node` declares a placeholder for a type drawn from some outside context; its intended use involves submsumption and/or sharing constraints.

Earlier, I claimed "modules are like vending machines", in the sense that they provide laws governing their interactions, isolating us from complex internal representations. The signature $$\mathsf{TraverseGraph}~\mathsf{with}~\mathsf{type}~\mathsf{node}=\mathsf{int}$$, on the other hand, doesn't isolate us from anything. Yet it still has laws that govern our interaction with it. So in this sense, it may be viewed as a degenerate vending machine that provides operations on existing datatypes without encapsulating any new datatypes of its own. Then, subsuming into `TraverseGraph` (as when we apply the `shortest_path` function) allows us to temporarily view `TraverseGraph.node` type as abstract in a local context.

# Conclusion

OCaml signatures may contain both opaque type declarations, which hide structs' type components behind abstractions, and transparent type declarations, which expose structs' type components in full to their clients. Transparent type declarations can be subsumed into opaque type declarations local contexts via signature subsumption. Sharing constraints allow us to strengthen a signature by replacing an opaque type component with a transparent one.

I will demonstrate all these concepts more in the next post, which introduces *functors*. A functor is essentially a function from modules to modules. Intuitively, it implements vending machines in terms of vending machines.