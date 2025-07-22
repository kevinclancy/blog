---
layout: page
title: Kevin Clancy - Software Engineer
permalink: /portfolio/
---

# Recent Work Experience

## CertiK

At CertiK, I used Python to develop an abstract interpretation tool on top of the [Slither](https://github.com/crytic/slither) static analyzer for Solidity code. This tool performed both points-to and integer range analyses. The former was used as a basis for taint tracking and many other ad-hoc bug detection tools. The latter was used for detecting integer overflows, division by zero, and unsafe casts. After a year at CertiK, I was promoted to Senior Software Engineer. My deep knowledge of abstract interpretation and order theory made me effective in this role.

## Amazon

At Amazon, I developed a CDK project (i.e. infrastructure deployment definition) for an internal web application for tracking releases of Amazon brand devices such as Kindles. I also performed a migration of the device database from an older Amazon-internal database system to DynamoDB. Additionally, I performed maintenance on the business logic that decides when to show Amazon Prime advertisements to customers. After a year at Amazon, I was recruited by CertiK.

# Personal Projects

I'm almost always working on programming projects to improve my software engineering abilities. These rarely ever get finished. Here are a few example projects, chosen mainly for recency.

## Game Kitchen - Lua IDE and Type Checker

Types are the most powerful tool I know of for making software more maintainable. When attending the University of Iowa, I read the book *Types and Programming Languages* by Benjamin Pierce. I applied the knowledge I gained about type systems by designing and implementing an optional type system for Lua. I also created a Lua debugger and an IDE called [Game Kitchen](https://bitbucket.org/kevinclancy/game-kitchen/src), targeted at the Lua game programming framework Love2D. Game Kitchen was written in F#, C#, C++, and Lua.

This is an extremely old project. But I've included it because it's one of the more ambitious projects I've worked on, and my software engineering sensibilities haven't changed much since then. I still believe that functional programming and design-by-contract are important techniques that lead to readable, maintainable software.

The following video is a bit tedious. I don't recommend watching the whole thing, but you can watch a random point in the video for a couple of minutes to see Game Kitchen in action.

<iframe width="420" height="315" src="https://www.youtube.com/embed/Ls3Z3xyKTC8" frameborder="0" allowfullscreen></iframe>

## Schema Types - NoSQL Database Schema Language

Before working at Amazon, I worked at a company called Epic Systems, which develops health record software for hospitals. Epic's software is based on the [MUMPS](https://en.wikipedia.org/wiki/MUMPS) database system. Unlike SQL, the MUMPS databse system provides no formal language for specifying database schemas. Therefore, at Epic, schemas are either conveyed informally in a company manual or internal wiki, or they are never defined at all. A formal schema language allows us to automatically generate validation routines, and it prevents common errors such as *coherence* errors where a single value may be interpreted in multiple ways.

[Schema Types](https://github.com/kevinclancy/SchemaTypes) is an experimental schema system for MUMPS, using type syntax to define the structure of MUMPS databases. It expresses the logical layer of the database using a language similar to the calculus of constructions. It expresses the physical layer using something akin to a standard type system, but which references the logical layer using indexed types. I used F# to create a prototype type-checker and test generation tool for Schema Types.

## Toy Compilers

After getting laid off from CertiK in October 2023, I took the opportunity to fill in some gaps in my knowledge of the foundations of computer science. I've long been interested in the theoretical side of programming languages, i.e. lambda calculi and type systems. However, until recently, I didn't know much about compilers. To rectify this, I've been reading some compilers books.

#### *The Essence of Compilation* by Jeremy Siek

I went through the first 6 chapters, [implementing a Lisp-to-x86 compiler](https://github.com/kevinclancy/EssentialsOfCompilation) in Racket, which includes:
* Imperative variables
* Loops
* Dataflow based liveness analysis
* Register allocation
* Dynamically-allocated Tuples
* A Two-Space Copying Garbage Collector

The garbage collector runtime was provided by the book.

#### *Compiler Design: Virtual Machines* by Reinhard Wilhelm and Helmut Seidl

Having already implemented the [virtual machine for a C-like language](https://github.com/kevinclancy/VirtualMachineCompiler) described in Chapter 2, I implemented the [virtual machine for an OCaml-like language](https://github.com/kevinclancy/MaMaCompiler/tree/cbv-variants-modules) described in Chapter 3. This includes both call-by-need and call-by-value versions (in different branches) with the following features
* Higher-order functions
* Curried Application
* Mutable reference cells
* Closures
* Mutually recursive functions
* Tuples
* Algebraic datatypes & match expressions (with 'when' clauses but no pattern matching)
* Tail call elimination

Note that the above virtual machines are "toy" virtual machines that piggyback off of the dotnet runtime. They don't implement their own garbage collectors and memory management. But see the "Virtual Machine for OCaml-style language" section below, where I implemented my own memory management and garbage collection for another implementation of this VM using Rust instead of F#.

## OCaml Projects

#### "The loop" server

[The loop server](https://github.com/kevinclancy/theloop-backend) is a process that responds to http requests by reading to and writing from a PostgreSQL database. Its behavior is partially documented in its [integration test](https://github.com/kevinclancy/theloop-backend/blob/main/test/test_theloop.ml), though this test needs to be extended with more detail. I was considering making Facebook Groups style website using this as the backend, but then decided to abandon it.

## Rust Projects

I love languages that feature sum types, because they make it easy to enforce invariants using static type checking, allowing me to make invalid states unrepresentable. Unfortunately, such languages have traditionally suffered from tooling and performance issues. I recently read through the official Rust book, discovering a language which has sum types, great tooling, and great performance. Even more exciting is Rust's borrow checker, which makes resource management both safe and efficient.

#### Ray Tracing in One Weekend

After reading the Rust book, I wanted to undertake a simple project that would give me a feel for writing my own Rust programs. I decided to implement the project described by Steve Hollasch's free online book "Ray Tracing in One Weekend". I didn't quite complete the book, getting only through chapter 7. This was a great opportunity for me to brush up on my linear algebra and 3D math skills.

[Here is the code](https://github.com/kevinclancy/raytracing-rust)

![Image]({{ site.baserul }}/assets/images/raytracer.png)

#### Virtual Machine for OCaml-style language

I've also created a [Rust implementation](https://github.com/kevinclancy/MaMaRust) of the virtual machine described above. This was a great opportunity to learn about low-level aspects of virtual machine implementation, which are not covered in Wilhelm and Seidl's book. In particular, I wrote my own memory management and a two-space copying garbage collector for this implementation.

# Links

[Linkedin](https://www.linkedin.com/in/kevin-clancy-740b13189/)

[Github](https://github.com/kevinclancy)