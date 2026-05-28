---
layout: post_classic
title: "The Semantics of Lens Logic"
date: 2026-05-03 09:00 -0700
categories: [modelling-computer-games]
subseries: logic
published: false
---

$$\newcommand{vrt}[2]{\left ( \begin{array}{l} #1 \\ #2 \end{array} \right )} $$
$$\newcommand{defeq}{\overset{\mathit{def}}{=}}$$

# Introduction

A subset $$A \subseteq X$$ can be read as a property of elements of $$X$$: an element belongs to $$A$$ if and only if it has the property. For example, the set $$\mathsf{even} = \{0, 2, 4, 6, \ldots\} \subseteq \mathbb N$$ is the property "is an even natural number", and $$2 \in \mathsf{even}$$ records the fact that $$2$$ has it.

Under this reading, set-theoretic operations correspond to logical ones. Union is "or": a natural number is even or odd if it lies in $$\mathsf{even} \cup \mathsf{odd} = \{0, 2, 4, \ldots\} \cup \{1, 3, 5, \ldots\}$$. Intersection is "and"; complement is "not"; the empty set is falsehood; the whole set $$X$$ is truth.

Above, we've used sets to model logic formulas and set-theoretic operations to model logical operations. This modelling is useful partly because we understand it intuitively. It helps us understand why the building blocks of logic have the algebraic properties they do. Furthermore, a formal correspondence between set theoretic semantics and logic allows us to justify the logic as sound.

I'm searching for a logical system to reason about dynamical systems and lenses. I'm not sure what such a logic would look like, and I haven't spent much time thinking about it. In this post, as a first step toward finding such a logic, I'll attempt to develop its underlying set theoretic semantics.

I will focus on partial systems, and the tic-tac-toe system from [Partial Systems]({% post_url 2025-10-25-partial-systems %}) in particular.

# Preliminary notions

Before we start, we establish some preliminary concepts. Namely, we formalize semantic notions for the sorts of systems that can be placed inside and outside an arena, respectively.

> **Definition**
>
> A **semantic partial system**, or **semantic system** for short, with input set $$A$$ and output set $$B$$ is a triple $$p \defeq (S, s_0 \in S, \bar{p}~: S \to (1 + S)^A \times B)$$, where $$S$$ is a set whose elements are the states the system can take on, $$s_0 \in S$$ is the initial state, and the function $$\bar{p}$$ maps each state $$s$$ to a pair $$(f, b)$$, where $$b \in B$$ is the output the system exposes at state $$s$$, and $$f$$ is the system's update function: for each input $$a \in A$$, $$f(a)$$ either "fails", if $$f(a) = \kappa_1 \ast$$, or produces a successor state $$s'$$ if $$f(a) = \kappa_2 s'$$. We write $$\mathsf{Sys}_{A, B}$$ for the collection of all semantic systems with input set $$A$$ and output set $$B$$.

Note that a semantic partial system is essentially a partial dynamical system paired with an initial state, as the function $$\bar{p} : S \to (1 + S)^A \times B$$ can be thought of as the pair of functions of types $$S \to B$$ and $$S \to (1 + S)^A$$, the latter of which can be desugared to $$S \times A \to (1 + S)$$.

## Example: Echo boxes

Recall that a dynamical system was defined as a lens of type $$\vrt{\mathsf{State}}{\mathsf{State}} \leftrightarrows \vrt{\mathsf{In}}{\mathsf{Out}}$$. Consider the domain arena $$\vrt{\mathsf{State}}{\mathsf{State}}$$; this arena mediates around some inner system that our initial formulation of dynamical systems had no way to describe. Ask yourself: how should this inner system be defined? The answer is that it is mediating an *echo box* which starts with some initial value $$s_0 \in \mathsf{State}$$ and produces (or "echoes") at each time step the value it received from its environment on the previous timestep. We define this inner system as:

$$\mathsf{echo}_{\mathsf{State}}^{s_0} \defeq (\mathsf{State}, s_0, \overline{\mathsf{echo}}_\mathsf{State})$$

where

$$\overline{\mathsf{echo}}_{\mathsf{State}}(s) \defeq (f, s)$$

where for all $$s' \in \mathsf{State}$$ we have

$$f(s') \defeq \kappa_2 s'$$

We will proceed to develop a more refined notion of a dynamical system than our original formulation: as a first approximation, it will consist of a lens $$\vrt{f^\sharp}{f} : \vrt{\mathsf{State}}{\mathsf{State}} \leftrightarrows \vrt{\mathsf{State}}{\mathsf{State}}$$ paired with an initial state $$s_0 \in \mathsf{State}$$ and a "condition" $$Q$$ on outer systems on $$\vrt{\mathsf{In}}{\mathsf{Out}}$$, with the expectation that $$\vrt{f^\sharp}{f}$$ "connects" an echo box $$\mathsf{echo}_{\mathsf{State}}^{s_0}$$ to some outer system satisfying the constraint $$Q$$.

# Predicates on arenas

> **Definition**
>
> An **inner system** $$p$$ on an arena $$\vrt{A}{B}$$ is a semantic system of the form $$(S, s_0 \in S, \bar{p} : S \to (1 + S)^A \times B)$$. An **outer system** on $$\vrt{A}{B}$$ is a semantic system of the form $$(S, s_0 \in S, \bar{p} : S \to (1 + S)^B \times A)$$.

An arena $$\vrt{A}{B}$$ can be thought of as a boundary between two semantic systems. An inner system $$p = (S, s_0, \bar{p} : S \to (1 + S)^A \times B)$$ and an outer system $$q = (T, t_0, \bar{q} : T \to (1 + T)^B \times A)$$. (Note the reversal of $$A$$ and $$B$$.) It witnesses a sequence of time steps. A time step with inner state $$s$$ and outer state $$t$$ consists of two phases:

1. The element $$b \defeq \bar{p}(s);\pi_2 \in B$$ flows from the inner system to the arena while simultaneously, the element $$a \defeq \bar{q}(t);\pi_2$$ flows from the outer system to the arena.

2. A new inner system state $$s'$$ is computed as $$\kappa_2 s' = (\bar{p}(s);\pi_1)(a)$$ and simultaneously a new outer system state $$t'$$ is computed as $$\kappa_2 t' = (\bar{q}(t);\pi_1)(b)$$. Alternatively, if either $$(\bar{p}(s);\pi_1)(a) = \kappa_1 \ast$$ or $$(\bar{q}(t);\pi_1)(b) = \kappa_1 \ast$$ then an error has occurred, and so the outer and inner system both grind to a halt; this isn't supposed to happen.

We use a set by selecting an element. A predicate on a set $$X$$ is a subset of $$X$$ that provides us partial information about a use of $$X$$ by constraining the set of possible elements to select.

Analogously, a useful notion of a predicate on an arena might provide partial information about a usage of an arena. But just how are arenas used? There are a few candidates for how to use an arena $$\vrt{A}{B}$$, among them are:

1. Selecting a semantic system $$p \defeq (S, s_0 \in S, \bar{p} : S \to (1 + S)^A \times B)$$ to place inside the boundary mediated by the arena.

2. Selecting a semantic system $$q \defeq (T, t_0 \in T, \bar{q} : T \to (1 + T)^B \times A)$$ to place outside the boundary mediated by the arena.

We develop a semantics in which both placements — inner and outer — give rise to predicate notions, and these interact via lens-mediated structure.

> **Definition**
>
> An **inner predicate** $$P$$ on an arena $$\vrt{A}{B}$$ is a set of inner systems on $$\vrt{A}{B}$$. An **outer predicate** $$Q$$ on an arena $$\vrt{A}{B}$$ is a set of outer systems on $$\vrt{A}{B}$$.

# Transforming arena predicates

Given an arena $$\vrt{A}{B}$$