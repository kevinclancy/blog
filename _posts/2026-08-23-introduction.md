---
layout: post_classic
title: "1. Modules are like Vending Machines"
date: 2026-08-23 09:00 -0700
categories: [understanding-module-systems]
keywords: module systems, ML modules, abstract types, signatures
published: true
---

Programmers often use dictionary datatypes to track associations between entities. A programmer types the following code
```
age_map.insert("joe", 20)
age_map.insert("bob", 30)
```
and understands that next time that their program encounters the expression `age_map.get("joe")` it will return the last value passed in as the second argument in `age_map.insert("joe", ____)`, namely `20` if no other calls to `insert` are made.

What the programmer does *not* need to understand is *why* invoking `age_map.get` reliably recalls `20`. The real reason is that `age_map` refers to a piece of data that is structured in a specific way that records associations. It may be implemented in terms of a hash table, or possibly a balanced binary tree. But those details are a distraction when trying to reason about real-world entities and high level business logic.

*Module systems* are a language feature that allow us to hide data representations such as hash tables and binary trees by "covering" them with constructs called *signatures*, which consist of the following components:

* A set of data types defining the sorts of entities that can be manipulated.
* A set of operations to be performed on the entities.
* A specification constraining what behaviors the signature's clients may observe when invoking a sequence of its operations.

For example, an `AgeMap` signature may define, among other operations, the following:
```
type t
val insert : t -> string -> int -> unit
val get : t -> string -> int
```
Above, `t` is the type of age maps, which the value `age_map` belongs to. And `unit` is the type of the empty tuple; it's returned by functions that are called only for their side effects.

In addition, the signature should also contain a behavior specification. Intuitively, it would list behavioral equations such as the following:
```
forall (map:t) (name:string) (age:int) (age2:int) (age3:int).
   (insert map name age) &&
   ((not (insert map name age2)) Until (age3 = get map name)) =>
   age3 = age
```

Above is a "pseudocode specification", in that it doesn't conform to any formally defined specification language that I know of. It says that if we insert `(name,age)` into the map and then do not insert any other age corresponding to `name` before calling `get map name`, then `get map name` should return `age`. More concisely, an entry inserted into the map stays there unless we override it.

The reason the above specification is pseudo-code rather than formal is that there is no widely used specification language for module signatures. Some have been developed; the most modern one that I know of is Gospel for OCaml. However, it's not clear that any are production ready. There are a variety of reasons why signature specifications aren't built in to modern module systems. Among them, deciding conformance to such specifications is undecidable.

Nonetheless, it should be obvious that specifications are a fundamental aspect of signatures. In modern software, they're present informally, as careful comments and field names which, like "map", reference commonly understood concepts. It's a mistake to explain module systems without acknowledging this, in the same way that it's a mistake to explain functions without acknowledging that they may have preconditions and postconditions that are more precise than a type system can express.

# Example - Functional Queues

## The functional queue signature

To make this more concrete, here is a signature written in OCaml. This signature describes a datatype for queues with functional updates. Instead of a single abstract datatype, we define a family of abstract datatypes indexed by the type `'a` of elements contained in the queues.

```
module type Queue = sig
  (** The standard functional queue ADT *)

  type 'a t
  (** A functional queue whose elements have type ['a] *)

  val empty : 'a t
  (** An empty queue *)

  val push : 'a t -> 'a -> 'a t
  (**
    [q' = push q v] pushes element [v] onto the back of queue [q] to
    obtain [q']
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

This signature describes *functional* queues in particular, meaning that each of the operations leave their argument queues intact. For example, `push q v` returns a new queue that behaves as `q` but with an additional element `v` pushed to its back; at the same time, we still have access to `q` even after calling `push` and binding its result to a variable `q'`.

This signature can be thought of as a *vocabulary*, declaring *types* that classify the objects or entities of the vocabulary (or "values" in programming terminology), as well as declaring *operations* that can be used to manipulate the entities. Above, we can see that OCaml's type declarations start with the keyword `type` and that its operation declarations start with the keyword `val`. However, importantly, the signature should also define *laws* constraining how the operations behave across multiple calls. Since OCaml has no built-in capacity to express such laws, I've included the laws as comments that begin with the text "LAW". Note that the laws are somewhat redundant with respect to the comments that have already been attached to values. In practice, laws typically are inferred from the val fields names and the comments attached to them, not from standalone comments labelled with "LAW".

## Implementing Signatures using Structs

While the signature describes how functional queues can be used, we cannot provide access to functional queues until we have *implemented* the signature by defining a struct. A struct can be thought of as a list of definitions of two kinds:

* For each type declaration in the signature, the struct must have a matching concrete type definition.
* For each val declaration in the signature, the struct must have a concrete value definition (a let binding) whose type matches the declaration's.

Here is an example implementation of the Queue signature:
```
module ListQueue = struct
  type 'a t = 'a list

  let empty : 'a t = []

  let is_empty (q : 'a t) : bool =
    match q with [] -> true | _ -> false

  let push (q : 'a t) (v : 'a) : 'a t = q @ [ v ]

  let pop (q : 'a t) : 'a * 'a t =
    match q with
    | [] -> invalid_arg "pop: empty queue"
    | a :: s -> (a, s)
end

module Q : Queue = ListQueue
```

It provides a family of concrete representations `'a list` for the abstract type family `'a t`. Then, it provides concrete values implementing each of the `Queue` signature's values.

The `: Queue` on the bottom line seals the `Queue` signature over the `ListQueue` struct. This first checks that the types of the `ListQueue` struct's let bindings match the `Queue` signature's val declarations. Then, it "creates a barrier" so that any client making use of `Q` cannot treat a value of type `'a Q.t` as a value of type `'a list` by pattern matching it into a cons cell or a nil list. Instead, all its clients can do is manipulate values of type `'a Q.t` using the vocabulary provided by `Queue`: namely, the `empty` value and the `push`, `pop`, and `is_empty` operations.

Note that the struct `ListQueue` is not an efficient queue implementation, because its `push` operation is O(n). A benefit of sealing is that when replacing the current implementation struct with a more efficient one, we need only check that the new implementation satisfies the `Queue` signature once. We do not need to check whether each individual client of `Q` is compatible with the new implementation.

## Using Sealed Structs

Once a struct has been sealed, we've obtained the "final product" of module systems: a "vocabulary" comprising a set of opaque types paired with a set of operations that can be performed on them, obeying a set of laws defined by the signature. The only thing left is to use this vocabulary. There are essentially two ways to do this:

* 1 - Use the sealed module to implement a function.
* 2 - Use the sealed module to construct another (higher level) sealed module.

## Using modules to implement functions

Here, I will demonstrate type 1 in depth.

One of the most well-known applications of the Queue abstract datatype is to implement the standard "breadth-first search" (or "BFS" for short) algorithm. This algorithm takes the node `source` of a graph as input and computes the shortest path from source to every other node in the graph.

In addition to access to the Queue datatype, the BFS algorithm requires access to the nodes and edges of a graph. For this, we define another signature:

```
module type TraverseGraph = sig
  (** An undirected graph of location nodes, only some of which are traversable.
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

Now we can use the BFS algorithm to implement the `shortest_path` function, which uses BFS in service of computing a shortest path (if any) from a source node to a destination node:
```
let shortest_path (module T : TraverseGraph)
                  (source : T.node) (dest : T.node) : T.node list option =
(**
  Computes a shortest path from `source` to `dest`, or returns `None`
  if no such path exists.

  # Parameters

  * T - The graph to traverse

  * source - The source node to begin from

  * dest - The destination node to end at

  # Return

  * A shortest path as a list of nodes [start, n1, n2, ..., dest].

  * Or None if no path from `source` to `dest` exists

*)

  let module NodeSet = Set.Make (struct
    type t = T.node
    let compare = T.compare_node
  end) in

  let module NodeMap = Map.Make (struct
    type t = T.node
    let compare = T.compare_node
  end) in

  let visited = ref NodeSet.empty in
  (** The set of nodes that have already been visited *)

  let parents = ref NodeMap.empty in
  (** Maps each node to its parent in the bfs tree *)

  let to_visit = ref (Q.push Q.empty source) in
  (** A queue of nodes to be visited *)

  while not (Q.is_empty (!to_visit)) do
    let (node, q') = Q.pop !to_visit in
    to_visit := q';
    match NodeSet.mem node !visited with
    | true ->
      ()
    | false ->
      visited := NodeSet.add node !visited;
      let should_enqueue (neighbor : T.node) : bool =
        not (NodeSet.mem neighbor !visited)
      in
      let traversable_neighbors = List.filter should_enqueue (T.adjacent_nodes node) in
      let add_parent (parents' : T.node NodeMap.t) (neighbor : T.node) : T.node NodeMap.t =
        match NodeMap.mem neighbor parents' with
        | true ->
          parents'
        | false ->
          NodeMap.add neighbor node parents'
      in
      parents := List.fold_left add_parent !parents traversable_neighbors;
      to_visit := List.fold_left (fun q n -> Q.push q n) !to_visit traversable_neighbors
  done;

  let rec construct_path (curr_node : T.node) (path_to_dest : T.node list) : (T.node list) option =
    (**

      # Parameters

      * curr_node - A node in the graph

      * path_to_dest - A path from `curr_node` to `dest`

      # Return

      * If it exists, a path from the `source` to `dest` node, including both source and destination

      * Otherwise, None

    *)
    match T.compare_node source curr_node with
    | 0 ->
      Some(curr_node :: path_to_dest)
    |_ ->
      match NodeMap.find_opt curr_node !parents with
      | Some(parent) ->
        construct_path parent (curr_node :: path_to_dest)
      | None ->
        None
  in
  construct_path dest []
```

Note that while the above function takes the `TraverseGraph` module as an argument `T`, the `Queue` module is referenced via a module identifier `Q` in context, as a "global". The reason for the distinction is that any two modules implementing the `Queue` signature are observationally indistinguishable, so we only need one implementation in our program. On the other hand, different `TraverseGraph` implementations may have distinct set of nodes and edges, and so distinct calls to `shortest_path` can take distinct `T` modules implementing `TraverseGraph` as arguments.

## Faulty sealing

Those familiar with the BFS algorithm know it depends on a loop invariant that only holds if the `Q` module is actually a queue; i.e., `Q` must satisfy all of the axioms of the `Queue` signature. Unfortunately, OCaml's type checker only ensures that a sealed struct provides all abstract types listed in the signature and that its fields' types agree with the types of the sealing signature's val declarations. It does not check that the struct satisfies the signatures' laws (obviously, since OCaml doesn't even allow us to express the laws formally).

To see how this can go wrong, consider that we can seal a struct implementing the standard "stack" abstract datatype using the `Queue` signature that we've already defined:

```
module ListStack = struct
  type 'a t = 'a list

  let empty = []

  let is_empty s = match s with [] -> true | _ -> false

  let push s v = v :: s

  let pop s =
    match s with
    | [] -> invalid_arg "pop: empty stack"
    | a :: s' -> (a, s')
end

module Q : Queue = ListStack
```

Because `ListStack` does not satisfy the "First-In-First-Out" law that queues do, the BFS loop invariant breaks, making `shortest_path` a faulty function that can return non-minimal paths between its `source` and `dst` arguments.


# Using modules to implement modules

## Implementing modules uing global modules

A long running process could implement a batch scheduler that executes tasks submitted from an external source at a regular interval. For fairness, we want to perform the earliest submitted tasks first, so we store them in a queue data structure.

Here is a rough sketch of what this would look like:
```
module Q : Queue = struct
  .. the list-based queue struct defined above ..
end

module type JobQueue = sig
   type t

   val create : time_delta -> t
   (** [create dt] returns a new job queue that executes pending tasks whenever at least time [dt] has passed since the last
       task execution *)

   val try_task : t -> bool
   (** Tries to execute the task at the front of the queue, returning `false` if the queue is empty or it is not yet time to execute the next task,
       or `true` if a task was executed
    *)

   val submit_task : t -> (unit -> unit) -> unit
   (** [submit_task b task] adds [task] to [b]  *)
end

module JQ : JobQueue = struct
   type t = {
      task_queue : (unit -> unit) Q.t ;
      (** A queue of "tasks", which are functions taking no input and producing no result *)

      last_executed : time ;
      (** The time that the queue last executed a task *)

      execution_period : time_delta
      (** The time minimum time required between executing distinct tasks *)
   } ref

   let try_task = ... implementation goes here ...

   let submit_task = ... implementation goes here ...
end
```

Both the types and value bindings in `JQ` can be defined in terms of types and terms of the `Q` module: we can see for example, that the JQ struct's type `t` is defined in terms of the type `Q.t`. Inside the implementation of `try_task`, which takes a value of type `t` as an argument, we may access the `Q.t` value contained therein, which we can only manipulate using the `Queue` signature.

## Implementing modules using parameter modules

Here `JQ` is defined in reference to `Q`, a specific implementation of the `Queue` signature. Because `Queue` provides a full specification of a module's behavior, there is no reason we would need to use multiple distinct `Queue` implementatations in the same program. However, we could weaken the `Queue` signature into a more general signature `PushPopBag` that does not enforce any ordering, satisfied by both the queue struct and the stack struct, as well as an implementation which allows `pop` to produce a randomly chosen element from the set.

```
module type PushPopBag = sig
  (** A bag which we can insert and delete elements from *)

  type 'a t
  (** A push-pop bag whose elements have type ['a] *)

  val empty : 'a t
  (** An empty bag *)

  val push : 'a t -> 'a -> 'a t
  (**
    [b' = push b v] pushes element [v] into the bag [b] to
    obtain [b']
  *)

  val pop : 'a t -> 'a * 'a t
  (** [(v, b') = pop b] Pops an arbitrary element [v] of the bag [b], yielding [b']

      ## Requires

      * [not (is_empty b)]

      ## Returns

      * v:'a - The element popped from [b]

      * b':'a t - The bag resulting from popping a value off [b]
  *)

  val is_empty : 'a t -> bool
  (** [is_empty b] Is bag [b] empty? *)

  (* LAW:
    The bag underlying the bag named "empty" (defined above) is the empty bag
  *)

  (* LAW:

    If c is the bag of elements of type 'a underlying a push-pop bag b then
    [is_empty b] is true iff c is the empty bag

  *)

  (* LAW:

     If c is the bag of elements underlying a push-pop bag b,
     then the bag underlying [push b a] is c + a (a inserted into c)

  *)

  (* LAW:

     If c is the bag of elements underlying a push-pop bag b, then [pop b] = (a, b'), where the bag underlying push-pop bag b' is (c - a),
     i.e. c with one occurrence of a removed.
  *)
end
```

Then, we could weaken the signature "job queue" to "job bag".
```
module type JobBag = sig
   type t

   val create : time_delta -> t
   (** [create dt] returns a new job bag that executes pending tasks whenever at least time [dt] has passed since the last
       task execution *)

   val try_task : t -> bool
   (** Tries to execute some task in the bag, returning `false` if the bag is empty or it is not yet time to execute the next task,
       or `true` if the task was executed.
    *)

   val submit_task : t -> (unit -> unit) -> unit
   (** [submit_task b task] adds [task] to [b]  *)
end
```

If only we had some "module function" construct that transforms module arguments into result modules, we could invoke it multiple times to produce different JobBag modules, all of which implement the `JobBag` signature but which have different push/pop behaviors (one with FIFO behavior, one with LIFO behavior, etc.).

Such "module functions" exist in OCaml. They are called *functors*. A functor `JB` for producing `JobBag` implementations might appear as follows.
```
module JB (B : PushPopBag) : JobBag = struct
   type t = r{
      task_bag : (unit -> unit) B.t ;
      (** A bag of "tasks", which are functions taking no input and producing no result *)

      last_executed : time ;
      (** The time that the queue last executed a task *)

      execution_period : time_delta
      (** The time minimum time required between executing distinct tasks *)
   } ref

   let try_task = ... implementation goes here ...

   let submit_task = ... implementation goes here ...
end
```

There's a lot more that can be said about functors, but I will leave that discussion for a later post.

# Conclusion: Modules are Like Vending Machines

I will wrap things up with a physical world analogy. Modules are like soda vending machines. The consumers interacts with the vending machine by performing a sequence chosen from the following set of operations:

* Coin insertion
* Product selection
* Coin return

Coin insertion, for example, requires a coin and a vending machine as input, and produces a modified vending machine as output. The consumer knows that if she inserts a coin and then selects a product, she will probably receive a soda. It is not enough for the machine to merely provide the three operations listed above; it must obey laws that are mutually understood by the machine's creator and the consumer (its signature). For example, the consumer knows that if she performs "coin insertion" and then performs "coin return" immediately afterward, the machine should emit the inserted coin. However, the consumer does not care about where her coin is stored in between, the shape and volume of its container, etc. These details are hidden from the consumer to make her life easier.

So far, we have used comments to express a signature's laws. In my next post, I will explore more formal techniques for doing so.