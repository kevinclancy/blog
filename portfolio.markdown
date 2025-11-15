---
layout: page
title: Kevin Clancy - Software Engineer
permalink: /portfolio/
---

# Kevin Clancy - Software Engineer

I'm a software engineer looking for opportunities in web development, program analysis, IDE development, and compiler development.

I have experience in C++, Java, Typescript, and Python, and for personal projects, I like to use F#, OCaml, and Rust. I believe in using static type checking, writing demanding functions with strong preconditions and postconditions, and using sum datatypes to make illegal states unrepresentable.

# Recent Experience

## CertiK

At CertiK, I used Python to develop an abstract interpretation tool on top of the Slither static analyzer for Solidity code. This tool performed both points-to and integer range analyses. The former was used as a basis for taint tracking and many other ad-hoc bug detection tools. The latter was used for detecting integer overflows, division by zero, and unsafe casts. Accoring to activity in CertiK's custom github branch, this tool was used as recently as Summer 2025. After a year at CertiK, I was promoted to Senior Software Engineer. My deep knowledge of abstract interpretation and order theory made me effective in this role.

## Amazon

At Amazon, I developed a CDK project (i.e. infrastructure deployment definition) for an internal web application for tracking releases of Amazon brand devices such as Kindles. I also performed a migration of the device database from an older Amazon-internal database system to DynamoDB. Additionally, I performed maintenance on the business logic that decides when to show Amazon Prime advertisements to customers. After a year at Amazon, I was recruited by CertiK.

# Personal Projects

I'm almost always working on programming projects to improve my software engineering abilities. These rarely ever get finished. Here are a few example projects, chosen mainly for recency.

## Web Development

### Brokenjaw.net message board ([github](github.com/kevinclancy/kboard))

**Skills:** React, Rust, Typescript, HTML, CSS, Loco, SQLite, Docker, Github Actions, AWS, Claude Code

I recently created an online message board for discussing jaw fractures, called [BrokenJaw.net](https://brokenjaw.net). This site uses the Rust-based backend framework Loco and the SQLite database system. Its frontend is written in Typescript and uses the React framework. I studied up on HTML, CSS, and React while developing this site, though the AI generated frontend code doesn't keep a clean separation of logical HTML layout and visual CSS styling, opting instead for inline styling. It uses github actions to build, test, and deploy docker containers to an AWS EC2 instance each time code is pushed to its github repo.

This website, while currently functional, is a work in progress. I plan to continue refining it over the coming months.

This is the first project that I've created using AI-assisted programming tools. In fact, much of the code was written using Claude Code. On one hand, Claude Code has an almost super-human ability to account for complex software contexts while synthesizing new code. On the other hand, it struggles to use advanced software engineering techniques like sum datatypes and pre and post-condition assertions. I use Claude Code to quickly generate a rough draft, quickly synthesizing the correct set of functions to call, even if they are poorly documented. Then, I revise the rough draft for correctness and readability. Claude Code is a powerful tool that I will continue to use for personal projects in the future.

### "The loop" server ([github](https://github.com/kevinclancy/theloop-backend))

**Skills:** OCaml, Dream, LWT, PostgreSQL

The loop server is a process that responds to http requests by reading to and writing from a PostgreSQL database. It was written in OCaml using the Dream web framework. Its behavior is partially documented in its [integration test](https://github.com/kevinclancy/theloop-backend/blob/main/test/test_theloop.ml), though this test needs to be extended with more detail. I was considering making Facebook Groups style website using this as the backend, but then decided to abandon it.

## Languages and Compilers

### Game Kitchen - Lua IDE and Type Checker ([bitbucket](https://bitbucket.org/kevinclancy/game-kitchen/src))

**Skills:** C#, F#, Lua, C++, Type Systems, IDE Development

Types are the most powerful tool I know of for making software more maintainable. When attending the University of Iowa, I read the book *Types and Programming Languages* by Benjamin Pierce. I applied the knowledge I gained about type systems by designing and implementing an optional type system for Lua. I also created a Lua debugger and an IDE called Game Kitchen, targeted at the Lua game programming framework Love2D. Game Kitchen was written in F#, C#, C++, and Lua.

This is an extremely old project. But I've included it because it's one of the more ambitious projects I've worked on, and my software engineering sensibilities haven't changed much since then. I still believe that functional programming and design-by-contract are important techniques that lead to readable, maintainable software.

The following video is a bit tedious. I don't recommend watching the whole thing, but you can watch a random point in the video for a couple of minutes to see Game Kitchen in action.

<iframe width="420" height="315" src="https://www.youtube.com/embed/Ls3Z3xyKTC8" frameborder="0" allowfullscreen></iframe>

### Toy Compilers and VMs

#### *The Essence of Compilation* by Jeremy Siek ([github](https://github.com/kevinclancy/EssentialsOfCompilation))

**Skills:** Compilers, Racket, x86 Assembly, Dataflow Analysis

I went through the first 6 chapters of *The Essence of Compilation*, implementing a Lisp-to-x86 compiler in Racket, which includes:
* Imperative variables
* Loops
* Dataflow based liveness analysis
* Register allocation
* Dynamically-allocated Tuples
* A Two-Space Copying Garbage Collector

The garbage collector runtime was provided by the book. (Though I created a similar garbage collector from scratch in the ``Rust-based Maurer Machine'' project.)

#### *Compiler Design: Virtual Machines* by Reinhard Wilhelm and Helmut Seidl ([github](https://github.com/kevinclancy/MaMaCompiler/tree/cbv-variants-modules))

**Skills:** F#, Virtual Machines

Having already implemented the [virtual machine for a C-like language](https://github.com/kevinclancy/VirtualMachineCompiler) described in Chapter 2, I implemented the ``Maurer Machine'' [virtual machine for an OCaml-like language](https://github.com/kevinclancy/MaMaCompiler/tree/cbv-variants-modules) described in Chapter 3. This includes both call-by-need and call-by-value versions (in different branches) with the following features
* Higher-order functions
* Curried Application
* Mutable reference cells
* Closures
* Mutually recursive functions
* Tuples
* Algebraic datatypes & match expressions (with 'when' clauses but no pattern matching)
* Tail call elimination

Note that the above virtual machines are "toy" virtual machines that piggyback off of the dotnet runtime. They don't implement their own garbage collectors and memory management.

#### Rust-based Maurer Machine ([github](https://github.com/kevinclancy/MaMaRust))

**Skills:** Rust, Virtual Machines, Garbage Collection

I've also created a Rust implementation of the virtual machine described above. This was a great opportunity to learn about low-level aspects of virtual machine implementation, which are not covered in Wilhelm and Seidl's book. In particular, I wrote my own memory management and a two-space copying garbage collector.

### Schema Types - NoSQL Database Schema Language ([github](https://github.com/kevinclancy/SchemaTypes))

**Skills:** F#, Type Systems, Database Schemas, MUMPS

Before working at Amazon, I worked at a company called Epic Systems, which develops health record software for hospitals. Epic's software is based on the [MUMPS](https://en.wikipedia.org/wiki/MUMPS) database system. Unlike SQL, the MUMPS databse system provides no formal language for specifying database schemas. Therefore, at Epic, schemas are either conveyed informally in a company manual or internal wiki, or they are never defined at all. A formal schema language allows us to automatically generate validation routines, and it prevents common errors such as *coherence* errors where a single value may be interpreted in multiple ways.

Schema Types is an experimental schema system for MUMPS, using type syntax to define the structure of MUMPS databases. It expresses the logical layer of the database using a language similar to the calculus of constructions. It expresses the physical layer using something akin to a standard type system, but which references the logical layer using indexed types. I used F# to create a prototype type-checker and test generation tool for Schema Types.

## Miscellaneous Projects

#### Ray Tracing in One Weekend ([github](https://github.com/kevinclancy/raytracing-rust))

**Skills:** Rust, Linear Algebra, 3D Math

After reading the Rust book, I wanted to undertake a simple project that would give me a feel for writing my own Rust programs. I decided to implement the project described by Steve Hollasch's free online book "Ray Tracing in One Weekend". I didn't quite complete the book, getting only through chapter 7. This was a great opportunity for me to brush up on my linear algebra and 3D math skills.

<img src="/assets/images/raytracer.png">

# Links

[Linkedin](https://www.linkedin.com/in/kevin-clancy-740b13189/)

[Github](https://github.com/kevinclancy)