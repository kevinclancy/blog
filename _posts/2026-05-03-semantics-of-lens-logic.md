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

Note that a semantic partial system is essentially a partial dynamical system paired with an initial state, as the function $$\bar{p} : S \to (1 + S)^A \times B$$ can be thought of as the pair of functions of types $$S \to B$$ and $$S \to (1 + S)^A$$, the latter of which can be uncurried to $$S \times A \to (1 + S)$$.

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

* We have $$\zeta(f) = (h, b)$$
* $$b$$ is any element of $$B$$
* For every $$a \in \varphi_A$$, $$h(a) = \kappa_2 f'$$ for some $$f' \in Z_{A,B}$$
* We have $$f' \in \upsilon$$

And the formula $$\downarrow \varphi_B$$ denotes the set of all behaviors $$f$$ such that $$f(\langle \rangle) \in \varphi_B$$.

# Transforming arena predicates

Fable wrote this section.

[#transforming-arena-predicates](#transforming-arena-predicates)

Given an arena $$\vrt{A}{B}$$, we now know two kinds of predicates on it: inner predicates, which constrain what may be placed inside the boundary it mediates, and outer predicates, which constrain what may be placed outside. But predicates stranded on individual arenas only take us so far. The whole point of our framework is that lenses relate arenas to one another, so a semantics of lens logic should explain how predicates travel along lenses.

We have a familiar prototype to imitate. A function $$h : X \to Y$$ induces two operations on ordinary predicates. The **direct image** operation transports a predicate on $$X$$ forward along $$h$$:

$$h_\ast\, P \defeq \{ h(x) \mid x \in P \}$$

and the **inverse image** operation transports a predicate on $$Y$$ backward along $$h$$:

$$h^\ast\, Q \defeq \{ x \in X \mid h(x) \in Q \}$$

A partial lens is not a function between arenas — an arena is a pair of sets rather than a set, and a lens passes information in two directions at once — so it is not immediately clear how to imitate this prototype. The key observation is that a partial lens induces a function, not between its domain and codomain arenas themselves, but between the collections of inner systems inhabiting them. Direct and inverse image along that induced function then come for free.

## Pushing inner systems forward

[#pushing-inner-systems-forward](#pushing-inner-systems-forward)

Throughout this section, fix a partial lens

$$\ell \defeq \vrt{f^\sharp}{f} : \vrt{A}{B} \leftrightarrows \vrt{C}{D}$$

so that its passforward and passback functions have types $$f : B \to D$$ and $$f^\sharp : B \times C \to 1 + A$$.

Suppose we place an inner system $$p$$ on $$\vrt{A}{B}$$ and then fit the lens $$\ell$$ around it like a shell. Viewed from outside the shell, the assembly behaves as an inner system on $$\vrt{C}{D}$$. At each time step it exposes an output in $$D$$, obtained by translating $$p$$'s output $$b \in B$$ downstream through $$f$$. It receives an input $$c \in C$$, which is translated upstream through $$f^\sharp(b, c)$$ before being delivered to $$p$$. And it can fail in two ways: the translation itself may fail — that is, $$f^\sharp(b, c) = \kappa_1 \ast$$, a precondition violation belonging to the lens — or the translation may succeed with some $$a \in A$$ on which $$p$$'s own update function fails. Formalizing this picture:

> **Definition**
>
> Let $$p \defeq (S, s_0, \bar{p})$$ be an inner system on $$\vrt{A}{B}$$. The **pushforward** of $$p$$ along $$\ell$$, written $$\ell_\ast\, p$$, is the inner system $$(S, s_0, \bar{q})$$ on $$\vrt{C}{D}$$ defined as follows: for each state $$s \in S$$, letting $$(u, b) \defeq \bar{p}(s)$$,
>
> $$\bar{q}(s) \defeq (v, f(b))$$
>
> where for each $$c \in C$$
>
> $$v(c) \defeq \begin{cases}
> \kappa_1 \ast & \text{if } f^\sharp(b, c) = \kappa_1 \ast \\
> u(a) & \text{if } f^\sharp(b, c) = \kappa_2\, a
> \end{cases}$$

Only the boundary behaviour changes: the state set and initial state pass through untouched, because the lens is a stateless adapter. Note also how two distinct failure modes are folded into one, just as they were when we composed partial lenses in [Partial Systems]({% post_url 2025-10-25-partial-systems %}): a precondition violation anywhere in the pipeline halts everything.

Pushforward lets us redeem the promise made in the echo box example. Consider a partial dynamical system — a partial lens $$\vrt{f^\sharp}{f} : \vrt{\mathsf{State}}{\mathsf{State}} \leftrightarrows \vrt{\mathsf{In}}{\mathsf{Out}}$$ — together with a chosen initial state $$s_0 \in \mathsf{State}$$. Pushing the echo box forward along it, we compute, for each $$s \in \mathsf{State}$$:

$$\overline{\left(\vrt{f^\sharp}{f}_{\!\ast}\; \mathsf{echo}^{s_0}_{\mathsf{State}}\right)}(s) = \left(f^\sharp(s, {-}),\; f(s)\right)$$

since the echo box's update function never fails and returns whatever state it is handed, so that the second case in the definition of $$v$$ collapses to $$f^\sharp(s, c)$$ itself. This is exactly the semantic system that intuition demands the pair $$(\vrt{f^\sharp}{f}, s_0)$$ denote: at state $$s$$ it exposes the output $$f(s)$$, and it evolves — or fails — according to $$f^\sharp$$. In slogan form, *a dynamical system is a lens wrapped around an echo box*, and pushforward is what "wrapped around" means. For instance, the tic-tac-toe environment of [Partial Systems]({% post_url 2025-10-25-partial-systems %}), started from the state $$(\mathit{ReceiveFrom}(0), b_0)$$ where $$b_0$$ is the empty board, denotes the inner system

$$\vrt{\mathit{nextState}_{\mathit{Environment}}}{\mathit{output}_{\mathit{Environment}}}_{\!\ast}\; \mathsf{echo}^{(\mathit{ReceiveFrom}(0),\, b_0)}_{\mathit{State}_{\mathit{Environment}}}$$

on the arena $$\vrt{1 + \mathit{Loc}}{\mathit{Board} \times \mathbf{3}}$$.

Pushforward also interacts well with lens composition. Write $$\mathsf{id} : \vrt{A}{B} \leftrightarrows \vrt{A}{B}$$ for the identity partial lens, whose passforward is the identity function on $$B$$ and whose passback is $$(b, a) \mapsto \kappa_2\, a$$.

> **Proposition**
>
> Pushforward is functorial. That is, for every inner system $$p$$ on $$\vrt{A}{B}$$ and all partial lenses $$\ell : \vrt{A}{B} \leftrightarrows \vrt{C}{D}$$ and $$m : \vrt{C}{D} \leftrightarrows \vrt{E}{F}$$, we have
>
> $$\mathsf{id}_\ast\, p = p \qquad \text{and} \qquad (m \circ \ell)_\ast\, p = m_\ast\, (\ell_\ast\, p)$$

Both equations follow by unfolding definitions. The second deserves a moment of attention: its two sides fail in exactly the same circumstances, because the composite passback defined in [Partial Systems]({% post_url 2025-10-25-partial-systems %}) signals a violation exactly when either constituent passback does. The bookkeeping we did when composing partial lenses is precisely what makes pushforward respect composition. Practically speaking, functoriality means it doesn't matter whether we wrap a system in several adapters one at a time or fuse the adapters into a single lens first.

## Direct and inverse image

[#direct-and-inverse-image](#direct-and-inverse-image)

With pushforward in hand, our partial lens $$\ell$$ induces a function

$$\ell_\ast : \mathsf{Sys}_{A, B} \to \mathsf{Sys}_{C, D}$$

from the inner systems on its domain arena to the inner systems on its codomain arena. Inner predicates are subsets of these collections, so we may transport them along $$\ell_\ast$$ exactly as in the prototype at the top of this section.

> **Definition**
>
> Let $$P$$ be an inner predicate on $$\vrt{A}{B}$$ and let $$Q$$ be an inner predicate on $$\vrt{C}{D}$$. The **direct image** of $$P$$ along $$\ell$$ is the inner predicate on $$\vrt{C}{D}$$ defined as
>
> $$\ell_\ast\, P \defeq \{ \ell_\ast\, p \mid p \in P \}$$
>
> and the **inverse image** of $$Q$$ along $$\ell$$ is the inner predicate on $$\vrt{A}{B}$$ defined as
>
> $$\ell^\ast\, Q \defeq \{ p \in \mathsf{Sys}_{A, B} \mid \ell_\ast\, p \in Q \}$$

We overload $$\ell_\ast$$ to denote both the pushforward of a single system and the direct image of a predicate; the argument always disambiguates. Functoriality of pushforward immediately gives $$(m \circ \ell)_\ast\, P = m_\ast (\ell_\ast\, P)$$ and $$(m \circ \ell)^\ast\, Q = \ell^\ast (m^\ast\, Q)$$ — note how inverse image reverses the order of composition.

Each operation has a natural reading. The direct image $$\ell_\ast\, P$$ holds of exactly those systems obtainable by wrapping some $$P$$-system in $$\ell$$: it is the most precise claim we can make about the outside of the shell knowing only that $$P$$ holds inside. Dually, the inverse image $$\ell^\ast\, Q$$ holds of a system just when wrapping it in $$\ell$$ produces a $$Q$$-system: it is the least demanding condition we can impose inside the shell that guarantees $$Q$$ outside. Readers with a program verification background may recognize a strongest postcondition and a weakest precondition here, with the lens playing the role of the program.

## A Galois connection

[#a-galois-connection](#a-galois-connection)

"Most precise" and "least demanding" are two descriptions of a single relationship.

> **Proposition**
>
> For every inner predicate $$P$$ on $$\vrt{A}{B}$$ and every inner predicate $$Q$$ on $$\vrt{C}{D}$$,
>
> $$\ell_\ast\, P \subseteq Q \quad \text{ if and only if } \quad P \subseteq \ell^\ast\, Q$$

Both sides assert the same thing: that $$\ell_\ast\, p \in Q$$ for every $$p \in P$$. A pair of monotone maps related by an equivalence of this shape is called a **Galois connection**; direct image is its *lower adjoint* and inverse image its *upper adjoint*. Two standard consequences, obtained by instantiating the equivalence at $$Q \defeq \ell_\ast\, P$$ and $$P \defeq \ell^\ast\, Q$$ respectively, are

$$P \subseteq \ell^\ast (\ell_\ast\, P) \qquad \qquad \ell_\ast\, (\ell^\ast\, Q) \subseteq Q$$

Once we have a logic with lens modalities, we expect this equivalence to reappear as a two-way proof rule for moving a lens from one side of an entailment to the other.

The introduction proposed union, intersection, and complement as the semantic counterparts of "or", "and", and "not". Our two predicate transformers treat these connectives very differently:

- Inverse image, being a preimage, commutes with *all* of them: $$\ell^\ast (Q_1 \cap Q_2) = \ell^\ast\, Q_1 \cap \ell^\ast\, Q_2$$, and likewise for unions, complements, the empty predicate, and the total predicate. In logical terms, $$\ell^\ast$$ behaves like a substitution, distributing freely through the propositional structure of any formula it is applied to.
- Direct image preserves unions and the empty predicate — a lower adjoint preserves all joins — but in general neither intersections nor complements. For intersections we have only the inclusion $$\ell_\ast (P_1 \cap P_2) \subseteq \ell_\ast\, P_1 \cap \ell_\ast\, P_2$$, which can be strict: a system on $$\vrt{C}{D}$$ may arise both as the pushforward of a member of $$P_1$$ and as the pushforward of a *different* member of $$P_2$$ without arising from any member of $$P_1 \cap P_2$$.
The lopsided behaviour of direct image is not a defect; it is the signature of an existential. For an ordinary function $$h$$, direct image is the semantic $$\exists$$, and it participates in a triple of adjoints $$h_\ast \dashv h^\ast \dashv \forall_h$$ whose rightmost member is the semantic $$\forall$$. The analogous operation exists in our setting too:

$$\forall_\ell\, P \defeq \{ q \in \mathsf{Sys}_{C, D} \mid \text{for all } p \in \mathsf{Sys}_{A, B}, \text{ if } \ell_\ast\, p = q \text{ then } p \in P \}$$

We won't need $$\forall_\ell$$ for a while, but its presence is an early hint that when we eventually design the logic's syntax, lenses should give rise to both diamond-like and box-like modalities.

## What about outer predicates?

[#what-about-outer-predicates](#what-about-outer-predicates)

Everything in this section so far concerns inner predicates, and one might expect a mirror-image story for outer predicates: surely a partial lens $$\ell : \vrt{A}{B} \leftrightarrows \vrt{C}{D}$$ should transform an outer system on $$\vrt{C}{D}$$ into an outer system on $$\vrt{A}{B}$$, by fitting the shell around the *hole* rather than around the occupant?

Surprisingly, it should not — at least not in this formalism. Consider an outer system $$q \defeq (T, t_0, \bar{q} : T \to (1 + T)^D \times C)$$ on $$\vrt{C}{D}$$ and attempt to construct from it an outer system on $$\vrt{A}{B}$$. Such a system must expose, at each of its states, an output in $$A$$. But the only means $$\ell$$ provides for producing an element of $$A$$ is the passback $$f^\sharp(b, c)$$, whose first argument is the current output of the inner system — a value that, in the two-phase time step of [Predicates on arenas](#predicates-on-arenas), is produced simultaneously with, and independently of, the outer system's own output. An outer system must compute its output from its own state alone, so the construction is stuck. (It goes through only in the special case where $$f^\sharp$$ ignores its first argument.)

Lenses therefore act on the inhabitants of arenas asymmetrically: inner systems push forward, but outer systems do not pull back. Rather than fight this asymmetry, we will let inner and outer predicates interact through the time-step protocol itself, via a satisfaction relation between the occupants of the two sides of an arena — the missing ingredient needed to make precise the "condition $$Q$$ on outer systems" from the echo box example. Defining that relation is our next task.
