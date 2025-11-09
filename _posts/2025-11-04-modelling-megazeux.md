---
layout: post_classic
title: Modelling MegaZeux
date: 2025-11-04 09:00 -0700
published: true
---

$$\newcommand{vrt}[2]{\left ( \begin{array}{l} #1 \\ #2 \end{array} \right )} $$
$$\newcommand{defeq}{\overset{\mathit{def}}{=}}$$

# Introduction

Now that [we've modeled tic-tac-toe using dynamical systems]({% post_url 2025-10-18-modelling-tic-tac-toe %}), let's try modelling MegaZeux. Our goal isn't to recreate every feature of the MegaZeux game creation system, but to understand its essence. Let's explore what MegaZeux is *trying* to be, redesigning any features we find to be arbitrary or inelegant.

## Concurrency, MegaZeux, and the Real World

In tic-tac-toe, players take turns making moves. But MegaZeux is a simulation of the real world. In the real world, all agents act concurrently.

As it turns out, MegaZeux does *not* faithfully reflect the concurrent nature of the real world. Robots execute in a turn-based manner every frame. The order that the robots execute in is decided by their position on the grid. It starts in the top-left corner, updating every robot it encounters as it moves right across the top row. Then it iterates across the second row, then the third, until it reaches the bottom-right corner of the game board.

This is awkward. It's an arbitrary rule that a game developer needs to learn to explain the behavior of MegaZeux games. Consider two side-by-side robots. If both attempt walking one cell to the left on the same frame then both robots move. However, if both walk one cell to the right then only the right robot moves, because when the left robot walks right, it crashes into the right robot.

Our model, unlike MegaZeux, will allow all robots to act at once. The game stepper will evolve in the following fashion. First, it executes the initialization step:

* The environment sends the initial game state to the robots; the game state includes positions of all robots, as well as wall positions.

Then, it executes these two steps in a loop:

1. All robots concurrently use their stored information, received from the environment on the previous step, to update their internal states and send their movement actions ($$\mathit{Idle}$$, $$\mathit{North}$$, $$\mathit{East}$$, $$\mathit{South}$$, $$\mathit{West}$$) to the environment.

2. The Environment executes the actions received from the robots to the best of its ability. If a robot proposes moving into a wall, off of the game board, or into a location where another robot currently is, then the move is not executed. If multiple robots attempt to move into the same cell then the environment randomly chooses one to move into the cell, committing the "losers" to their present positions. The environment sends the resulting state of the game board back to the robots, along with a boolean indicating to each robot whether its move succeeded.

## Non-deterministic lenses and dynamical systems

According to our plan, when multiple robots attempt moving into the same cell, we randomly choose one to succeed. But this requires a feature beyond the scope of the technique of lenses and dynamical systems described in the previous blog post.

We'll need to extend our definitions so that a dynamical system can update its state non-deterministically. Recall that if $$\mathit{X}$$ is a set then $$P X$$ is the set of all subsets of $$X$$, where $$P -$$ is called the powerset operator. If we think of a set as a collection of possibilities, then what we want is a new formulation of dynamical systems, where the passback function has the type

$$\mathit{State} \times \mathit{In} \to P \mathit{State}$$

Instead of producing a successor state as output, it produces the set of all possible successor states. In [Categorical Systems Theory](https://www.davidjaz.com/Papers/DynamicalBook.pdf), this is called a *possibilistic* system. Before we define possibilistic systems, we must first define a possibilistic version of the more general notion of a lens.

> **Definition**
>
> A **$$P$$-lens** $$\vrt{f^\sharp}{f} : \vrt{A^-}{A^+} \leftrightarrows \vrt{B^-}{B^+}$$ is a pair consisting of two functions:
> * A passforward function $$f : A^+ \to B^+$$
> * A passback function $$f^\sharp : A^+ \times B^- \to P A^-$$
>
> The arena $$\vrt{A^-}{A^+}$$ is called the **domain** of $$\vrt{f^\sharp}{f}$$ and the arena $$\vrt{B^-}{B^+}$$ is called the **codomain** of $$\vrt{f^\sharp}{f}$$.

Additionally, we have a nondeterminstic notion of dynamical systems.

> **Definition**
>
> A **possibilistic dynamical system** is a $$P$$-lens of the form
>
> $$\vrt{\mathit{nextState}}{\mathit{output}} : \vrt{\mathit{State}}{\mathit{State}} \leftrightarrows \vrt{\mathit{In}}{\mathit{Out}}$$
>
> That is, a dynamical system is a $$P$$-lens whose domain is an arena of the form $$\vrt{\mathit{State}}{\mathit{State}}$$ for some set $$\mathit{State}$$.

We must also update the definitions of our composition operators.

<!--
We must also update the definitions of our composition operators. They will make use of a few constructs related to the powerset operator. First, if $$f : X \to Y$$ is a function, then we define a function $$P f : PX \to PY$$ as

$$Pf(A) \defeq \{ f(x) \mid x \in A \}$$

In English, to apply $$Pf$$ to a subset $$A$$ of $$X$$, we apply $$f$$ to every element of $$A$$ and collect the results into a subset of $$Y$$.

There are also a few families of functions that we will need. First, for every set $$X$$ there is a function $$\eta_X : X \to PX$$ which maps an element $$x$$ to the single-element set containing $$x$$:

$$\eta_X(x) \defeq \{ x \}$$

Second, for every set $$X$$ there is a function $$\mu_X : PPX \to PX$$. It takes the union of a set of subsets of $$X$$:

$$\mu_X(Q) \defeq \bigcup Q$$

For example, if $$Q$$ is the set $$\{ A_1, A_2, A_3 \}$$ where each $$A_i$$ is a subset of $$X$$, then $$\mu_X(Q) = A_1 \cup A_2 \cup A_3$$. Now we are ready to define $$P$$-lens composition.
-->

> **Definition**
>
> Given $$P$$-lenses $$\vrt{f^\sharp}{f} : \vrt{A^-}{A^+} \leftrightarrows \vrt{B^-}{B^+}$$ and
> $$\vrt{g^\sharp}{g} : \vrt{B^-}{B^+} \leftrightarrows \vrt{C^-}{C^+}$$ their **composite**
> $$\vrt{g^\sharp}{g} \circ \vrt{f^{\sharp}}{f}$$ is defined as $$\vrt{h^\sharp}{h}$$, where
>
> * $$h$$ is defined as the function composite $$g \circ f$$
> * $$h^\sharp$$ is defined such that
>
>$$h^\sharp(a^+, c^-) \defeq \bigcup_{b^-~\in~g^\sharp(f(a^+), c^-)} f^\sharp(a^+, b^-)$$

We must also update our parallel composition operator.

> **Definition**
>
> Given two lenses $$\vrt{f^\sharp}{f} : \vrt{A^-}{A^+} \leftrightarrows \vrt{B^-}{B^+}$$ and
$$\vrt{g^\sharp}{g} : \vrt{C^-}{C^+} \leftrightarrows \vrt{D^-}{D^+}$$ we define their parallel
> product
> $$\vrt{f^\sharp}{f} \otimes \vrt{g^\sharp}{g} : \vrt{A^- \times C^-}{A^+ \times C^+} \leftrightarrows \vrt{B^- \times D^-}{B^+ \times D^+}$$ is defined as the lens
> $$\vrt{h^\sharp}{h}$$, where
>
> $$h(a^+, c^+) \defeq (f(a^+), g(c^+))$$
>
> and
>
> $$h^\sharp(a^+, c^+, b^-, d^-) \defeq \{ (a^-, c^-) \mid a^- \in f^\sharp(a^+, b^-),~c^- \in g^\sharp(c^+, d^-) \}$$

# The Model

We call the game world that our robots traverse the *board*.
Our MegaZeux model is parameterized by three natural numbers:

* n - the number of robots
* w - the width of the board in cells
* h - the height of the board in cells

Recall that we write a natural number $$k$$ in bold to denote the set of natural numbers that are less than it, i.e. $$\mathbf{k} \defeq \{ 0, \ldots, k-1 \}$$ .

The set of board cell locations is then $$\mathbf{w} \times \mathbf{h}$$. Our board state can then be defined as:

$$\mathit{BoardState} \defeq \mathbf{2}^{\mathbf{w} \times \mathbf{h}} \times (\mathbf{w} \times \mathbf{h})^\mathbf{n}$$

Our environment takes steps of two types: receive and send. For send steps, we need to store a mapping from robot identifiers to booleans indicating whether the robot's most recent action succeeded. Thus we define

$$\mathit{StepType} \defeq 1 + \mathbf{2}^{\mathbf{n}}$$

where the left summand $$1$$ is used for receive steps and the right summand is for send steps.

The state of our environment $$\mathit{Env}$$ is then defined as:

$$\mathit{State}_{\mathit{Env}} \defeq \mathit{BoardState} \times \mathit{StepType}$$

Elements of the first component $$\mathbf{2}^{\mathbf{w} \times \mathbf{h}}$$ are functions mapping each board cell location to $$0$$ if the cell is vacant and $$1$$ if the cell contains a wall. Elements of the second component $$(\mathbf{w} \times \mathbf{h})^{\mathbf{n}}$$ map each robot identifier $$i$$ to the cell location at which robot $$i$$ is currently located.

At each turn, a robot can act by either staying **I**dle or moving in one of the directions **N**orth, **E**ast, **S**outh, or **W**est. Therefore, we define the set of actions as:

$$\mathit{Act} \defeq \{ I, N, E, S, W \}$$

And the environment's input, if any, is a map from each robot identifier $$i$$ to the action submitted by robot $$i$$ on a turn.

$$\mathit{In}_{\mathit{Env}} \defeq (1 + \mathit{Act})^{\mathbf{n}}$$

Since the robots only submit actions at every other step, we also include the summand $$1$$ in the definition of $$\mathit{In}_{\mathit{Env}}$$ to use on turns where the robots do not submit.

On every $$\mathit{Send}$$ step, the environment must output to each robot a copy of the game state along with a boolean (an element of $$\mathbf{2}$$) indicating whether or not the robot's most recently submitted action succeeded.

$$\mathit{Out}_{\mathit{Env}} \defeq (1 + \mathit{BoardState} \times \mathbf{2})^{\mathbf{n}}$$

The top-down perspective of the MegaZeux game stepper then looks as follows.

<figure>
<img
  src="/assets/images/modelling-megazeux/stepper.drawio.png"
  style="margin-top: 30px; margin-bottom: 30px"
>
<figcaption>
Figure 1
</figcaption>
</figure>

Note that, unlike the tic-tac-toe game stepper, the MegaZeux stepper has no need for multiplexors and demultiplexors. This is because all robots act simultaneously at each turn.

## The Environment

Now that we've presented the structure of our possibilistic game stepper as a diagram, let's define the subsystem $$\mathit{Env}$$. It is defined as the possibilistic system

$$\vrt{\mathit{nextState}_{\mathit{Env}}}{\mathit{output}_{\mathit{Env}}} : \vrt{\mathit{State}_{\mathit{Env}}}{\mathit{State}_{\mathit{Env}}} \leftrightarrows \vrt{\mathit{In}_{\mathit{Env}}}{\mathit{Out}_{\mathit{Env}}}$$

Before defining the functions, we introduce a notational convenience. Recall that elements of $$\mathit{Out}_{\mathit{Env}} = (1 + \mathit{BoardState} \times \mathbf{2})^{\mathbf{n}}$$ are functions from robot identifiers (elements of $$\mathbf{n}$$) to output values. We can construct such a function using *lambda notation*: the expression $$\lambda i \in \mathbf{n} . (0, \ast)$$ denotes the constant function that always returns $$(0, \ast)$$ regardless of which robot identifier $$i$$ you pass in. Similarly, $$\lambda i \in \mathbf{n} . (1, (m, r, s(i)))$$ denotes the function that takes a robot identifier $$i$$ and returns $$(1, (m, r, s(i)))$$, where $$s(i)$$ is the success value for robot $$i$$. If you're familiar with programming, this is analogous to anonymous functions like Python's `lambda i: (0, star)` or JavaScript's `i => (0, star)`.

Now, the $$\mathit{output}_{\mathit{Env}}$$ function replicates the wall and robot positions to all robots and distributes to each robot the result of its last action:

$$\mathit{output}_{\mathit{Env}} : \mathbf{2}^{\mathbf{w} \times \mathbf{h}} \times (\mathbf{w} \times \mathbf{h})^{\mathbf{n}} \times (1 + \mathbf{2}^{\mathbf{n}}) \to (1 + \mathit{BoardState} \times \mathbf{2})^{\mathbf{n}}$$

$$\mathit{output}_{\mathit{Env}}(m, r, (0, \ast)) \defeq \lambda i \in \mathbf{n} . (0, \ast)$$

$$\mathit{output}_{\mathit{Env}}(m, r, (1, s)) \defeq \lambda i \in \mathbf{n} . (1, (m, r, s(i)))$$

The $$\mathit{nextState}_{\mathit{Env}}$$ passback function has type:

$$\mathit{nextState}_{\mathit{Env}} : \mathit{State}_{\mathit{Env}} \times \mathit{In}_{\mathit{Env}} \to P\mathit{State}_{\mathit{Env}}$$

Expanding the definitions:

$$\mathit{nextState}_{\mathit{Env}} : (\mathbf{2}^{\mathbf{w} \times \mathbf{h}} \times (\mathbf{w} \times \mathbf{h})^{\mathbf{n}} \times (1 + \mathbf{2}^{\mathbf{n}})) \times (1 + \mathit{Act})^{\mathbf{n}} \to P(\mathbf{2}^{\mathbf{w} \times \mathbf{h}} \times (\mathbf{w} \times \mathbf{h})^{\mathbf{n}} \times (1 + \mathbf{2}^{\mathbf{n}}))$$

We need a few helper functions. First, given a robot position map $$r$$ and an action $$a \in \mathit{Act}$$, we define $$\mathit{move}(r, i, a)$$ to compute the proposed new position for robot $$i$$:

$$\mathit{move} : (\mathbf{w} \times \mathbf{h})^{\mathbf{n}} \times \mathbf{n} \times \mathit{Act} \to \mathbf{w} \times \mathbf{h}$$

$$\mathit{move}(r, i, a) \defeq \begin{cases}
(x, y) & \text{if } a = I \\
(x, y - 1) & \text{if } a = N \\
(x + 1, y) & \text{if } a = E \\
(x, y + 1) & \text{if } a = S \\
(x - 1, y) & \text{if } a = W
\end{cases}$$

$$\text{where } (x, y) \defeq r(i)$$

Next, we need to determine if a move is valid (doesn't hit a wall, go out of bounds, or collide with a robot's current position):

$$\mathit{valid} : \mathbf{2}^{\mathbf{w} \times \mathbf{h}} \times (\mathbf{w} \times \mathbf{h})^{\mathbf{n}} \times \mathbf{n} \times \mathit{Act} \to \mathbf{2}$$

$$\mathit{valid}(m, r, i, a) \defeq \begin{cases}
\mathit{true} & \text{if } \mathit{move}(r,i,a) \in \mathbf{w} \times \mathbf{h} \\
& \land~m(\mathit{move}(r,i,a)) = 0 \\
& \land~\mathit{move}(r,i,a) \notin \{r(j) \mid j \in \mathbf{n}\} \\
\mathit{false} & \text{otherwise}
\end{cases}$$

Now we can define $$\mathit{nextState}_{\mathit{Env}}$$. On a Send step in which the environment's preconditions are satisfied (i.e., all robots provide no input), we transition back to Receive:

$$\mathit{nextState}_{\mathit{Env}}((m, r, (1, s)), \lambda i \in \mathbf{n} . (0, \ast)) \defeq \{(m, r, (0, \ast))\}$$

On a Receive step where the environment's preconditions are satisified (i.e., all robots provide an action), we process the robot actions and non-deterministically resolve conflicts. Let $$a \in (1 + \mathit{Act})^{\mathbf{n}}$$ be the input, where $$a(i) = (1, \alpha_i)$$ for all $$i \in \mathbf{n}$$ and some action $$\alpha_i$$.

For each cell location $$\ell \in \mathbf{w} \times \mathbf{h}$$, let $$\mathit{candidates}(\ell, m, r, a)$$ denote the set of robots attempting to move to $$\ell$$:

$$\mathit{candidates} : (\mathbf{w} \times \mathbf{h}) \times \mathbf{2}^{\mathbf{w} \times \mathbf{h}} \times (\mathbf{w} \times \mathbf{h})^{\mathbf{n}} \times \mathit{Act}^{\mathbf{n}} \to P\mathbf{n}$$

$$\mathit{candidates}(\ell, m, r, a) \defeq \{i \in \mathbf{n} \mid \mathit{valid}(m,r,i,a(i)) \land \mathit{move}(r,i,a(i)) = \ell\}$$

Then we define the set of target locations that robots are attempting to move to:

$$\mathit{targets} : \mathbf{2}^{\mathbf{w} \times \mathbf{h}} \times (\mathbf{w} \times \mathbf{h})^{\mathbf{n}} \times \mathit{Act}^{\mathbf{n}} \to P(\mathbf{w} \times \mathbf{h})$$

$$\mathit{targets}(m, r, a) \defeq \{ \ell \in \mathbf{w} \times \mathbf{h} \mid |\mathit{candidates}(\ell, m, r, a)| \geq 1 \}$$

> **Definition** For sets $$X$$ and $$Y$$, we write $$X \rightharpoonup Y$$ for the set of partial functions from $$X$$ to $$Y$$,
> defined as
>
> $$X \rightharpoonup Y \defeq \{ f \subseteq X \times Y \mid ((x,y) \in f \wedge (x,z) \in f) \Rightarrow y = z \} $$
>
> For $$f \in X \rightharpoonup Y$$, we write $$f(x) \! \downarrow$$ and say that "$$f$$ is defined at $$x$$" when there
> exists a $$y \in Y$$ with $$(x,y) \in f$$. We write $$f(x) \! \uparrow$$ and say that "$$f$$ is undefined at $$x$$" when no
> such $$y \in Y$$ exists. When $$f(x) \! \downarrow$$, we write $$f(x)$$ for the unique $$y \in Y$$ with $$(x,y) \in f$$.
>
> We define $$\mathit{dom}(f)$$, the domain of the partial function $$f : X \rightharpoonup Y$$, as
>
> $$\mathit{dom}(f) \defeq \{ x \mid f(x) \! \downarrow \} $$

For each target, a *resolution* chooses one of the candidates to successfully move into the target.

$$\mathit{resolutions} : \mathbf{2}^{\mathbf{w} \times \mathbf{h}} \times (\mathbf{w} \times \mathbf{h})^{\mathbf{n}} \times \mathit{Act}^{\mathbf{n}} \to P(\mathbf{w} \times \mathbf{h} \rightharpoonup \mathbf{n})$$

$$\mathit{resolutions}(m, r, a) \defeq \{ q \in \mathbf{w} \times \mathbf{h} \rightharpoonup \mathbf{n} \mid \mathit{dom}(q) = \mathit{targets}(m,r,a) \wedge \forall \ell \in \mathit{dom}(q).~q(\ell) \in \mathit{candidates}(\ell, m, r, a)  \}$$

Given a resolution $$q \in \mathit{resolutions}(m,r,a)$$, we compute the new robot positions and success indicators. The function $$\mathit{nextr}$$ computes the new position for each robot: a robot moves to its target cell if it won the resolution for that cell (i.e., the resolution chose it), otherwise it stays in place.

$$\mathit{nextr} : \mathit{Act}^{\mathbf{n}} \times (\mathbf{w} \times \mathbf{h})^{\mathbf{n}} \times (\mathbf{w} \times \mathbf{h} \rightharpoonup \mathbf{n}) \to (\mathbf{w} \times \mathbf{h})^{\mathbf{n}}$$

$$\mathit{nextr}(a,r,q)(i) \defeq \begin{cases}
\mathit{move}(r, i, a(i)) & \text{if } q(\mathit{move}(r, i, a(i))) = i \\
r(i) & \text{otherwise}
\end{cases}$$

The function $$\mathit{success}$$ indicates whether each robot's action succeeded: a robot succeeds if its position changed, or if it chose to idle.

$$\mathit{success} : \mathit{Act}^{\mathbf{n}} \times (\mathbf{w} \times \mathbf{h})^{\mathbf{n}} \times (\mathbf{w} \times \mathbf{h} \rightharpoonup \mathbf{n}) \to \mathbf{2}^{\mathbf{n}}$$

$$\mathit{success}(a,r,q)(i) \defeq \begin{cases}
1 & \text{if } \mathit{nextr}(a,r,q)(i) \neq r(i) \vee a(i) = I \\
0 & \text{otherwise}
\end{cases}$$

We can now define the Receive step transition in terms of these helper functions:

$$\mathit{nextState}_{\mathit{Env}}((m, r, (0, \ast)), a) \defeq \{ (m, \mathit{nextr}(a,r,q), (1, \mathit{success}(a,r,q))) \mid q \in \mathit{resolutions}(m, r, a) \}$$

For any other combination of state and input (all violations of our environment's preconditions), we return the empty set:

$$\mathit{nextState}_{\mathit{Env}}(z, a) \defeq \emptyset$$

## The Robots

For each robot identifier $$i \in \mathbf{n}$$, we write $$\mathit{Robot}_i$$ for the possibilistic system corresponding to robot $$i$$.

We posit the existence of a set $$S_i$$ containing the *mental state* of robot $$i$$. A value of $$S_i$$ might contain a program counter for the script robot $$i$$ is currently running, and it might contain private data members as well. For distinct $$i,j \in \mathbf{n}$$ the sets $$S_i$$ and $$S_j$$ are not required to be equal and are typically distinct.

In addition to mental state, the state of robot $$i$$ also contains *administrative state*. The administrative state $$S_{\mathit{admin}}$$, which is uniform across all robots, synchronizes the robot with the environment and stores stimuli received from the environment.

$$S_{\mathit{admin}} \defeq 1 + (\mathit{BoardState} \times 2)$$

The value $$(0,\ast) \in S_{\mathit{admin}}$$ means that the environment is currently in a $$\mathit{Submit}$$ state. This means that the robots are currently receiving the board state and their succes values. The value $$(1, (m,r,s)) \in S_{\mathit{admin}}$$ means that the environment is currently in a $$\mathit{Receive}$$ state. It stores the wall map $$m$$, robot map $$r$$, and success value $$s$$ transmitted from the environment on the previous step.

Then the state set of robot $$i$$ is defined as:

$$\mathit{State}_{\mathit{Robot}_i} \defeq S_{\mathit{admin}} \times S_i$$

Figure 1 above shows us the input and output sets shared by the $$n$$ robots:

$$\mathit{In}_{\mathit{Robot}_i} \defeq 1 + (\mathit{BoardState} \times 2)$$

$$\mathit{Out}_{\mathit{Robot}_i} \defeq 1 + (\mathit{Act})$$
