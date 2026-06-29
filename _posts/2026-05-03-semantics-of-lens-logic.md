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

# Predicates on arenas

## Inner and outer systems

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

As a first attempt, we might define our predicate notions as follows:

> **Preliminary definition**
>
> An **inner predicate** $$P$$ on an arena $$\vrt{A}{B}$$ is a set of inner systems on $$\vrt{A}{B}$$. An **outer predicate** $$Q$$ on an arena $$\vrt{A}{B}$$ is a set of outer systems on $$\vrt{A}{B}$$.

But this definition is problematic. An obvious predicate in $$\vrt{A}{B}$$ is $$\mathsf{true}$$, also called $$\top$$: the collection of all possible inner systems on $$\vrt{A}{B}$$. This collection, however, is so big that it is unwieldy. Since each triple in the collection contains a set as its first component, we have at least one inner system for each non-empty set. This implies that the collection of inner systems is "too large" to be a set.

## System behaviors

A better approach is to prohibit our logic from making any statements about a system's internal state. The $$S$$ component of an inner system $$(S, s_0, \bar{p} : S \to (1 + S)^A \times B)$$ can be thought of as its representation: the set of the system's possible internal states. Instead of quantifying over all possible representations, we choose a single representation $$Z$$ whose elements directly convey the *behavior* of the system:

Letting, $$A^*$$ denote the set of all finite sequences of elements of $$A$$, and letting $$\langle a_1, a_2, \ldots, a_n \rangle$$ denote the member of $$A^*$$ whose elements are, in order, $$a_1, a_2, \ldots a_n$$, we define:

$$Z_{A,B} \defeq \{ f \in (1 + B)^{A^*} \mid (\forall \sigma \in A^*.~f(\sigma) = \kappa_1 \ast \Rightarrow \forall \sigma' \in A^* f(\sigma \cdot \sigma') = \kappa_1 \ast)  \wedge f(\langle \rangle) \neq \kappa_1 \ast \} $$

$$f \in Z_{A,B}$$ represents a system such that, for $$\sigma \in A^*$$,

* If $$f(\sigma) = \kappa_2 b$$ then the system produces output $$b$$ after receiving the sequence of inputs $$\sigma$$.
* If $$f(\sigma) = \kappa_1 \ast$$ then the system halts due to a precondition violation after reading some prefix of the sequence of inputs $$\sigma$$ (possibly the entire sequence).

With the above points in mind, the two constraints in the definition of $$Z_{A,B}$$ can be understood as follows:

The first constraint

$$(\forall \sigma \in A^*.~f(\sigma) = \kappa_1 \ast \Rightarrow \forall \sigma' \in A^*.~f(\sigma \cdot \sigma') = \kappa_1 \ast) $$

can be understood to mean that once our system halts due to a precondition violation, it cannot produce any outputs after receiving further inputs. The second constraint

$$f(\langle \rangle) \neq \kappa_1 \ast$$

can be understood to mean that no precondition can be violated before the system has received any inputs. We summarize the above discussion with the following definition:

> **Definition**
>
> The set $$Z_{A,B}$$ defined above is called the set of **inner behaviors** on the arena $$\vrt{A}{B}$$.

> **Definition**
>
> A subset of $$Z_{A,B}$$ is called an **inner predicate** on the arena $$\vrt{A}{B}$$.

## Behaviors represent classes of systems

Any element $$f \in Z_\vrt{A}{B}$$ can serve as the initial state of an inner system $$(Z_{A,B}, f, \zeta : Z_{A,B} \to (1 + Z_{A,B})^A \times B)$$.

TODO: define and explain the final coalgebra

# A logic on inner predicates

Now we explore the design of our logic. First, we want the ability to constrain the set of inputs our inner systems expect at specific times, as well as constrain the set of outputs our inner systems produce. A constraint on a set like $$A$$ is essentially a subset $$X \subseteq A$$; thus, our logic will contain a sublogic for expressing subsets of the input and output sets $$A$$ and $$B$$. Such logics are not novel; this sublogic, formulas could denote subsets and logical operators such as $$- \wedge -$$ and $$- \vee -$$ could correspond to set-theoretic operators such as $$- \cap -$$ and $$- \cup -$$. We elide the definition of this sublogic for now, but use subscripted symbols such as $$\varphi_A, \psi_A$$ as metavariables for formulas expressing subsets of $$A$$, and likewise use $$\varphi_B, \psi_B$$ for formulas expressing subsets of $$B$$. As an abuse of notation, we write $$a \in \varphi_A$$ to mean that $$a$$ is an element of the set denoted by $$\varphi_A$$.

The metavariable $$\upsilon$$ is used for logic formulas describing sets of inner behaviors on $$\vrt{A}{B}$$. A first pass at its syntax might look as follows:

$$\upsilon ::= [\varphi_A](\upsilon) \mid \downarrow \varphi_B \mid \upsilon_1 \wedge \upsilon_2 \mid \upsilon_1 \vee \upsilon_2$$

Roughly, the formula $$[\varphi_A](\upsilon)$$ denotes the set of all behaviors $$f$$ such that:

* $$\zeta(f) = (h, b)$$
* $$b$$ is any element of $$B$$
* For every $$a \in \varphi_A$$, $$h(a) = \kappa_2 f'$$ for some $$f' \in Z_{A,B}$$
* $$f' \in \upsilon$$

And the formula $$\downarrow \varphi_B$$ denotes the set of all behaviors $$f$$ such that $$f(\langle \rangle) \in \varphi_B$$.
