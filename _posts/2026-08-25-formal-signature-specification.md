---
layout: post_classic
title: "2. Formal Signature Specification"
date: 2026-08-25 09:00 -0700
categories: [understanding-module-systems]
published: true
---

There are other ways to communicate a signature's laws besides informal comments. One is to use the formal specification language Gospel. Another is to pair each signature with a test suite.

# Gospel

The Gospel specification language is a research project. While it's not clear to me if it's ready to be used for large real-world projects, my experimentation with it has gone pretty well.

## Queue Example

Gospel can only be used to annotate signatures that occur in `.mli` files. We can adapt our functional queue signature to satisfy this requirement, creating a `queue.mli` file with the following content:
```
  (** The standard functional queue ADT *)

  type 'a t
  (** A functional queue whose elements have type ['a] *)

  val empty : 'a t
  (** An empty queue *)

  val push : 'a t -> 'a -> 'a t
  (**
    [q' = push q v] pushes element [v] onto queue [q] to
    obtain [q']
  *)

  val pop : 'a t -> 'a * 'a t
  (** [(a, q') = pop q] Pops front [a] of the queue [q], yielding [q']

      ## Requires

      * [non_empty q]

      ## Returns

      * a:'a - The element popped from the front of [q]

      * q':t - The queue resulting from popping the front off [q]
  *)

  val is_empty : 'a t -> bool
  (** [is_empty q] Is queue [q] empty? *)
```

Now let's rewrite the signature's comments using Gospel
```
(** The standard functional queue ADT *)

type 'a t
(** A functional queue whose elements have type ['a] *)
(*@
  model s: 'a sequence
*)

val empty : 'a t
(*@
  ensures empty.s = Sequence.empty
*)

val push : 'a t -> 'a -> 'a t
(*@
   q' = push q v
   ensures q'.s = (Sequence.singleton v) ++ q.s
*)

val pop : 'a t -> 'a * 'a t
(*@
  (v, q') = pop q
  requires q.s <> Sequence.empty
  ensures q'.s ++ (Sequence.singleton v) = q.s
*)

val is_empty : 'a t -> bool
(*@
  b = is_empty q
  ensures b <-> q.s = Sequence.empty
*)
```

As we can see, Gospel allows us to attach a modelling value to each abstract datatype:
```
type 'a t
(** A functional queue whose elements have type ['a] *)
(*@
  model s: 'a sequence
*)
```

We may supply one or many fields to model an abstract type. (For example, a *bounded* queue may require both a field of type `sequence` like we have, and also a field of type `int` to represent the queue's capacity.)
Gospel provides a [wide variety](https://ocaml-gospel.github.io/gospel/gospel/Gospelstdlib/index.html#sequences) of datatypes for us to use for model fields, including sequences, arrays, bags, sets, and references.

The annotation for each val field first names all of the function's arguments and results:
```
(v, q') = pop q
```

Then it lists a series of preconditions on lines beginning with the word `requires`
```
requires q.s <> Sequence.empty
```

Then it lists a series of postconditions on lines beginning with the word `ensures`
```
ensures q'.s ++ (Sequence.singleton v) = q.s
```

We can type-check Gospel specifications by running `gospel check queue.mli`.

Gospel has a few advantages over informal specifications. First, by virtue of being formal, it is unambiguous and readable. Different programmers may describe module behavior in comments using their own idiosyncratic ways of thinking. Other programmers reading the comments may therefore have difficulty interpreting them. There is no question of how to interpret a formal specification, so misunderstandings cannot arise.

Second, formal specifications can be consumed by tools that either test or verify them. I will proceed to describe a few such tools.

## Ortac Wrapper

Ortac Wrapper generates a wrapped version of a Gospel-specified module, which monitors the implementation against the specification as it executes, raising an error as soon as a discrepancy is encountered.

For each abstract datatype `t` and each model field its specification declares, the module implementor is required to write a *projection* function to project a value of type `t` to a value of the model field type. Such projection functions are annotated with a `projection_for` decorator:
```
val to_sequence : 'a t -> 'a sequence [@@projection_for s]
```
Note that Ortac Wrapper is a behind the most recent release of Gospel; it does not support fields of type `sequence`, so `list` must be used instead.

As long as the wrapped module doesn't raise a monitoring error, then it conforms to the same specified signature as the unwrapped module. The wrapped module could therefore be used instead of the unwrapped one during testing and/or debug builds where performance isn't important.

## Ortac QCheck-STM, Ortac Monolith

The QCheck-STM and Monolith plugins both generate a test suite targeting a Gospel specification. QCheck-STM seems overly fussy, as it rejected the `Queue` signature's `pop` method because the result type has a `Queue.t` nested inside a pair.

I didn't spend much time experimenting with these tools, but
the ability to automatically generate a test suite is extremely useful, so I should probably revisit them.

## Cameleer

Testing can prove the presence of errors, but not their absence. To do that, we need to perform formal reasoning. Cameleer uses an automated theorem prover to prove Gospel specifications correct. Unfortunately, it is not compatible with newer versions of OCaml, so I didn't test it.

# Hegel

Hegel is a property-based testing library. Instead of using Gospel for specification, we can test our signature's laws using a hand-written Hegel test. The advantages to this approach are that first, hand-written testing is more likely to be able to express our signature's laws, and second, Hegel seems more production ready than Gospel-related tools. The disadvantages are that hand-written tests are more likely to contain mistakes due to human error, and also that they are a less direct way of expressing properties and therefore potentially ambiguous.

I asked Claude to generate a functor that takes a module matching the `Queue` signature as input and produces a module providing a Hegel test which tests that the input module satisfies the `Queue` laws.The generated functor body defines two tests. First, the first tests the queue laws:
```
  type state =
    { q : int Q.t
    ; model : int list
    }

  let push_rule =
    Stateful.Rule.create ~name:"push" ~step:(fun tc st ->
      let v = draw tc (integers ~min_value:(-1000) ~max_value:1000 ()) in
      { q = Q.push st.q v; model = v :: st.model })

  let pop_rule =
    Stateful.Rule.create ~name:"pop" ~step:(fun tc st ->
      assume tc (st.model <> []);
      let v, q' = Q.pop st.q in
      match split_last st.model with
      | None -> assert false
      | Some (expected, rest) ->
        Alcotest.(check int) "pop returns the oldest element" expected v;
        { q = q'; model = rest })

  let is_empty_rule =
    Stateful.Rule.create ~name:"is_empty" ~step:(fun _tc st ->
      Alcotest.(check bool)
        "is_empty agrees with the model"
        (st.model = [])
        (Q.is_empty st.q);
      st)

  let%hegel_test queue_matches_model tc =
    Stateful.run
      ~init:{ q = Q.empty; model = [] }
      ~rules:[ push_rule; pop_rule; is_empty_rule ]
      ~sexp_of_state:(fun st -> sexp_of_model st.model)
      tc
```

This is what Hegel calls a *stateful* test. It performs a sequence of operations to a state. Each operation may assert correctness criteria on the state. Our Gospel specification already suggests the basic outline of the stateful test we want: define the state as a pair containing a `Queue.t` value and its model, creating one operation corresponding to each `Queue` operation, and when operations that "emit data" (the popped value, or `is_empty`'s boolean result), assert the equality of the data emitted from the `Queue.t` value and the model.

The sequence of operations is generated randomly, but it's able avoid (or filter out) operations that violate preconditions, using the `assume` function, as can be seen in the `pop_rule` rule.

Claude also generated a test to check that the queue operations are `persistent`; that is, calling `push` leaves the argument queue unchanged, and furthermore leaves all of the "ancestor" queues it was derived from via older operations unchanged as well:
```
  type pstate =
    { cur : int Q.t
    ; cur_model : int list
    ; olds : (int Q.t * int list) Stateful.Pool.t
    }

  let ppush =
    Stateful.Rule.create ~name:"push" ~step:(fun tc st ->
      let v = draw tc (integers ~min_value:(-1000) ~max_value:1000 ()) in
      (* remember the queue as it was BEFORE the push, with its model *)
      Stateful.Pool.add st.olds (st.cur, st.cur_model);
      { st with cur = Q.push st.cur v; cur_model = v :: st.cur_model })
  ;;

  let ppop =
    Stateful.Rule.create ~name:"pop" ~step:(fun tc st ->
      assume tc (st.cur_model <> []);
      Stateful.Pool.add st.olds (st.cur, st.cur_model);
      let _, q' = Q.pop st.cur in
      match split_last st.cur_model with
      | None -> assert false
      | Some (_, rest) -> { st with cur = q'; cur_model = rest })
  ;;

  let check_old =
    Stateful.Rule.create ~name:"observe_an_older_queue" ~step:(fun tc st ->
      let q_old, model_old = draw_silent tc (Stateful.Pool.values_reusable st.olds) in
      Alcotest.(check bool)
        "an older queue kept its emptiness"
        (model_old = [])
        (Q.is_empty q_old);
      (match split_last model_old with
       | None -> ()
       | Some (expected, _) ->
         let v, _ = Q.pop q_old in
         Alcotest.(check int) "an older queue kept its contents" expected v);
      st)
  ;;

  let%hegel_test queue_is_persistent tc =
    Stateful.run
      ~init:{ cur = Q.empty; cur_model = []; olds = Stateful.Pool.create tc }
      ~rules:[ ppush; ppop; check_old ]
      ~sexp_of_state:(fun st -> sexp_of_model st.cur_model)
      tc
  ;;
```

The model for this test includes a *pool* whose elements each pair an ancestor queue with the model matching that ancestor. The `Queue` signature's state-altering operations `push` and `pop` are included as `ppush` and `ppop`, which not only perform state updates but also insert the "old" states into the pool. Finally, a new operation `check_old` ensures that the immediately observable behavior (popped elements and is_empty) of a randomly selected ancestor queue matches that of its model.

When I ran these tests on the `ListQueue` struct, they passed as expected. When I ran them on the `ListStack` struct, they failed, producing this error message:
```
--- Failure: queue_matches_model (test/queue_test.ml:66) ---------------
Falsified after 10 test cases (0 discarded):

  state = ()
  Step 1: push
    draw_1 = 0
  state = (0)
  Step 2: push
    draw_2 = 1
  state = (1 0)
  Step 3: pop
ASSERT pop returns the oldest element

Exception: Alcotest assertion failure
File "test/queue_test.ml", line 55, character 8:
FAIL pop returns the oldest element

   Expected: `0'
   Received: `1'
```

It lists the shortest sequence of operation it can find that trips an assertion. It also displays all of the intermediate states and random data it generates along the way. After pushing two elements and then popping, we see that the stack popped the newest element, when according to the model it should have popped the oldest.

Hegel is polished, expressive, and concise, and compared to Gospel it's not *that* much more verbose. All in all, it's not a bad tool.

# Artificial Intelligence

Instead of checking signature conformance using testing or formal verification, why not try ask an AI coding tool to check it for us? This certainly requires less labor on our part. Normally I would ask Claude Code to perform a task like this, but out of curiosity, I decided to use the free Grok web interface. I provided it with the original, informal queue signature:
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
And a disguised version of the ListStack struct:
```
module ListQueue = struct
  type 'a t = 'a list

  let empty = []

  let is_empty s = match s with [] -> true | _ -> false

  let push s v = v :: s

  let pop s =
    match s with
    | [] -> invalid_arg "pop: empty queue"
    | a :: s' -> (a, s')
end
```

Here is Grok's answer:
```
No (not correctly / not as an ADT).
Type-wise, ListQueue matches the signature: the type 'a t and the four values have compatible types, so an OCaml compiler would accept module LQ : Queue = ListQueue.
Semantically it does not implement the specified queue ADT, because it violates the documented laws:

The laws require FIFO behaviour: push q a must append a to the end of the underlying sequence (s ++ [a]), and pop must remove from the front.
The implementation does the opposite:
push s v = v :: s prepends, and pop takes the head.
Consequently the structure is a stack (LIFO), not a queue.
After a sequence of pushes the order of elements is reversed relative to what the laws demand, so the “LAW” comments for push and pop are false.

(The empty/is_empty laws happen to hold, but that is not enough.)
A correct pure-list implementation of the given signature would need to append on push (or maintain the list in reverse order and reverse on pop, or use the classic two-list representation). The given ListQueue does none of those things.
```

Not bad. But to be fair, this was one of the easiest challenges I could give it. It would be interesting to try this on less stereotypical signatures some time, or to create a type checker that dispatches semantic checking tasks to an LLM.

# Conclusion

While natural language comments can be frustratingly vague, a signature's laws can be expressed in other ways. We can specify the laws using the Gospel specification language or through Hegel tests. Furthermore, there are formal verification, testing, and AI tools that can help us gain confidence that a struct actually satisfies a signatures laws--or demonstrate why it fails to satisfy them!

In my next post, I plan to discuss two things:

* How and why OCaml programs are structured as networks of module definitions.
* How the operations that a module provides may perform side effects such as state updates, and what the implications of that are.

Stay tuned.