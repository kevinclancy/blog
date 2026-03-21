---
layout: post_classic
title: "Interlude: Off the Rails"
date: 2025-10-25 09:00 -0700
categories: [modelling-computer-games]
published: false
---

$$\newcommand{vrt}[2]{\left ( \begin{array}{l} #1 \\ #2 \end{array} \right )} $$
$$\newcommand{defeq}{\overset{\mathit{def}}{=}}$$

# Precondition Violations

In [Modelling Tic-Tac-Toe]({% post_url 2025-10-18-modelling-tic-tac-toe %}), I defined the $$\mathit{StepType}$$ component of the environment's state as the set:

$$\{ \mathit{ReceiveFrom}(0), \mathit{SubmitTo}(1), \mathit{ReceiveFrom}(1), \mathit{SubmitTo}(0), \mathit{IllegalState} \}$$

When you read that post, you probably understood that $$\mathit{IllegalState}$$ is used for "off rails" behavior and the other four elements are used for "on rails" behavior. The environment enters $$\mathit{IllegalState}$$ if either

* It receives a move while it's in a $$\mathit{SubmitTo}$$ state
* It receives an "empty input" $$(0, \ast)$$ while it's in a $$\mathit{ReceiveFrom}$$ state
* It's already in $$\mathit{IllegalState}$$

The idea is that we have engineered our dynamical system such that none of the above three scenarios ever arises given that our initial state is valid. The set of conditions which are true for exactly those state/input pairs expected by our lens are called the lens's *preconditions*. Unexpected state/input pairs such as the three listed above are called *precondition violations*. The environment system's preconditions are

* Locations $$(1,\ell)$$ must only be provided as the first component of the input when in a $$\mathit{ReceiveFrom}$$ state
* An "Empty input" $$(0, \ast)$$ must only be provided as the first component of the input when in a $$\mathit{SubmitTo}$$ state
* The system must never be in $$\mathit{IllegalState}$$

Making both off rails and on rails states siblings, in the sense that they are elements of the same set $$\mathit{StepType}$$, is obfuscating. Off rails states should be distinguished from on rails states formally--not just with suggestive names such as $$\mathit{IllegalState}$$.

One benefit of making the distinction formal is that it unambiguously communicates to anyone examining our system the distinction between what our system is intended to do and what it is not intended to do. More importantly, it relieves us from the pointless and misleading task of defining the values that $$\mathit{nextState}_{\mathit{Environment}}$$ outputs when it is fed $$\mathit{IllegalState}$$ as the second component of its input.

We can achieve such a formal distinction by defining notions of *partial lenses* and *partial dynamical systems*.

# Partial Systems Theory

The notion of *partial lenses* extends that of lenses by including an innate encoding of lens preconditions.

> **Definition**
>
> A **partial lens** $$\vrt{f^\sharp}{f} : \vrt{A^-}{A^+} \leftrightarrows \vrt{B^-}{B^+}$$ is a pair consisting of two functions:
> * A passforward function $$f : A^+ \to B^+$$
> * A passback function $$f^\sharp : A^+ \times B^- \to 1 + A^-$$
>
> The arena $$\vrt{A^-}{A^+}$$ is called the **domain** of $$\vrt{f^\sharp}{f}$$ and the arena $$\vrt{B^-}{B^+}$$ is called the **codomain** of $$\vrt{f^\sharp}{f}$$.

From this, we derive a notion of partial dynamical systems.

> **Definition**
>
> A **partial discrete dynamical system** (or **partial dynamical system** for short) is a partial lens of the form
>
> $$\vrt{\mathit{nextState}}{\mathit{output}} : \vrt{\mathit{State}}{\mathit{State}} \leftrightarrows \vrt{\mathit{In}}{\mathit{Out}}$$
>
> That is, a partial dynamical system is a lens whose domain is an arena of the form $$\vrt{\mathit{State}}{\mathit{State}}$$ for some set $$\mathit{State}$$.

Taking this definition apart, we can see that a partial dynamical system is a pair of two functions:

$$\mathit{output} : \mathit{State} \to \mathit{Out}$$

and

$$\mathit{nextState} : \mathit{State} \times \mathit{In} \to 1 + \mathit{State}$$

The interesting part is the definition of $$\mathit{nextState}$$. Unlike the $$\mathit{nextState}$$ function for a plain dynamical system, this function does **not** map every state/input pair to a successor state. Instead, it maps those state/input that it expects to $$(1, s)$$ where $$s$$ is a successor state, and it maps those state/input pairs that it does not expect (i.e., those state/input pairs that violate the lens's precondition) to $$(0,\ast)$$.

Note the asymmetry in the "type signature" of $$\mathit{nextState}$$: a sum appears on the right side but not the left. This is because once a precondition has been violated, we choose not to continue executing our system. Defining the system's response to receiving input while in an illegal state would only distract and confuse.

We now provide the remaining components of our partial systems theory: standard composition and parallel composition.

> **Definition**
>
> Given partial lenses $$\vrt{f^\sharp}{f} : \vrt{A^-}{A^+} \leftrightarrows \vrt{B^-}{B^+}$$ and
> $$\vrt{g^\sharp}{g} : \vrt{B^-}{B^+} \leftrightarrows \vrt{C^-}{C^+}$$ their **composite**
> $$\vrt{g^\sharp}{g} \circ \vrt{f^{\sharp}}{f}$$ is defined as $$\vrt{h^\sharp}{h} : \vrt{A^-}{A^+} \leftrightarrows \vrt{C^-}{C^+}$$, where
>
> * $$h$$ is defined as the function composite $$g \circ f$$
> * $$h^\sharp$$ is defined such that $$h^\sharp(a^+, c^-) \defeq \begin{cases}
> (0, \ast) & \text{if } g^\sharp(f(a^+), c^-) = (0, \ast) \\
> (0, \ast) & \text{if } g^\sharp(f(a^+), c^-) = (1, b^-) \text{ and } f^\sharp(a^+, b^-) = (0, \ast) \\
> (1, a^-) & \text{if } g^\sharp(f(a^+), c^-) = (1, b^-) \text{ and } f^\sharp(a^+, b^-) = (1, a^-)
> \end{cases}$$

Intuitively, the above definition says that if either the inner or the outer lens signals a precondition violation during passback, then the composite lens signals a precondition violation as well.

Next, we define parallel composition.

> **Definition**
>
> Given two partial lenses $$\vrt{f^\sharp}{f} : \vrt{A^-}{A^+} \leftrightarrows \vrt{B^-}{B^+}$$ and
$$\vrt{g^\sharp}{g} : \vrt{C^-}{C^+} \leftrightarrows \vrt{D^-}{D^+}$$ we define their parallel
> product
> $$\vrt{f^\sharp}{f} \otimes \vrt{g^\sharp}{g} : \vrt{A^- \times C^-}{A^+ \times C^+} \leftrightarrows \vrt{B^- \times D^-}{B^+ \times D^+}$$ as the lens
> $$\vrt{h^\sharp}{h}$$, where
>
> $$h(a^+, c^+) \defeq (f(a^+), g(c^+))$$
>
> and
>
> $$h^\sharp(a^+, c^+, b^-, d^-) \defeq \begin{cases}
> (0, \ast) & \text{if } f^\sharp(a^+, b^-) = (0, \ast) \text{ or } g^\sharp(c^+, d^-) = (0, \ast) \\
> (1, (a^-, c^-)) & \text{if } f^\sharp(a^+, b^-) = (1, a^-) \text{ and } g^\sharp(c^+, d^-) = (1, c^-)
> \end{cases}$$

So intuitively, the above definition says that the precondition of the juxtaposition of two systems is satisfied exactly when both preconditions of the constituent systems are satisfied.

# Improving Tic-Tac-Toe with Partial Systems

Now let's redefine the Tic-Tac-Toe system more elegantly using our fancy new partial systems machinery. First, we remove the awkward $$\mathit{IllegalState}$$ element from our $$\mathit{StepType}$$ set:

$$\mathit{StepType} \defeq \{ \mathit{ReceiveFrom}(0), \mathit{SubmitTo}(1), \mathit{ReceiveFrom}(1), \mathit{SubmitTo}(0) \}$$

The environment's output function can then be defined as follows:

$$
\mathit{output}_\mathit{Environment}(t, b) \defeq \begin{cases}
((1, b), n) & \text{ if } t = SubmitTo(n) \\
((0, \ast), 2) & \text{ if } t = ReceiveFrom(n)
\end{cases}
$$

The above definition looks far more satisfactory than the one in my previous post, because it does not include the distracting case where $$t = \mathit{IllegalState}$$.

We must also update our definition of $$\mathit{nextState}_{\mathit{Environment}}$$.

$$~\\$$
$$\mathit{nextState}_{\mathit{Environment}} : \mathit{StepType} \times \mathit{Board} \times (1 + \mathit{Loc}) \to (1 + \mathit{StepType} \times \mathit{Board})$$
$$~\\$$
$$
\mathit{nextState}_{\mathit{Environment}}(ReceiveFrom(n), b, (1,\ell)) \defeq \begin{cases}
(1, (SubmitTo((n+1)~\%~2), b[\ell \mapsto n])) & \text{if } b(\ell) = \_ \\
(1, (SubmitTo((n+1)~\%~2), b)) & \text{otherwise}
\end{cases}
$$
$$~\\$$
$$
\mathit{nextState}_{\mathit{Environment}}(SubmitTo(n), b, (0,\ast)) \defeq (1, (ReceiveFrom(n), b))
$$
$$~\\$$
$$\text{and for any other } (s,b,z) \in \mathit{StepType} \times Board \times (1 + Loc) \text{ we define}$$
$$~\\$$
$$
\mathit{nextState}_{\mathit{Environment}}(s,b,z) \defeq (0, \ast)
$$

This is essentially the same as the definition in the previous post, with an important difference: $$\mathit{nextState}_{\mathit{Environment}}$$ maps arguments which satisfy the environment's precondition to outputs of the form $$(1,(s,b))$$ and maps arguments that violate the precondition to outputs of the form $$(0, \ast)$$. The difference between the two is a *formal* distinction between desirable behavior and undesirable behavior; that's just good mathematical hygiene. What's more, we didn't bother with the distracting and counterproductive task of defining how our system behaves after something has already gone wrong.

