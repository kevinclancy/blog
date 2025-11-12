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

* The environment sends the initial game state to the robots; the game state includes positions of all robots, as well as wall positions. The environment sends $$1$$ to each robot telling it that its last action succeeded, even though no such action exists; robots should be programmed to ignore this value. The robots use this information to update their states and store actions ($$\mathit{Idle}$$, $$\mathit{North}$$, $$\mathit{East}$$, $$\mathit{South}$$, $$\mathit{West}$$) to submit on the next step.

Then, it executes steps of these two types in a loop:

1. All robots concurrently transmit their stored action, computed on the previous step, to the environment. The robots also clear their stored actions. The environment executes the actions received from the robots to the best of its ability. If a robot proposes moving into a wall, off of the game board, or into a location where another robot currently is, then the move is not executed. If multiple robots attempt to move into the same cell then the environment randomly chooses one to move into the cell, committing the "losers" to their present positions. The environment stores the resulting state of the game board and booleans indicating whether each robot's most recent action succeeded to submit back to the robots on the next step.

2. (This is like the initialization step, but now the success values are meaningful.) The environment sends the current game state to the robots; the game state includes positions of all robots, as well as wall positions. The environment also sends to each robot a boolean indicating whether the robot's most recently submitted action succeeded. The robots use this information to update their states and store actions to submit on the next step.

## Non-deterministic lenses and dynamical systems

According to our plan, when multiple robots attempt moving into the same cell, we randomly choose one to succeed. But this requires a feature beyond the scope of the technique of lenses and dynamical systems described in the previous blog post. Namely, non-determinism.

We'll need to extend our definitions so that an environment can update its state non-deterministically. Recall that if $$\mathit{X}$$ is a set then $$P X$$ is the set of all subsets of $$X$$, where $$P -$$ is called the powerset operator. If we think of a set as a collection of possibilities, then what we want is a new formulation of dynamical systems, where the passback function has the type

$$\mathit{State} \times \mathit{In} \to P \mathit{State}$$

Instead of producing a successor state as output, it produces the set of all possible successor states. In [Categorical Systems Theory](https://www.davidjaz.com/Papers/DynamicalBook.pdf), this is called a *possibilistic* system. Before we define possibilistic systems, we must first define a possibilistic version of lenses.

> **Definition**
>
> A **$$P$$-lens** $$\vrt{f^\sharp}{f} : \vrt{A^-}{A^+} \leftrightarrows \vrt{B^-}{B^+}$$ is a pair consisting of two functions:
> * A passforward function $$f : A^+ \to B^+$$
> * A passback function $$f^\sharp : A^+ \times B^- \to P A^-$$
>
> The arena $$\vrt{A^-}{A^+}$$ is called the **domain** of $$\vrt{f^\sharp}{f}$$ and the arena $$\vrt{B^-}{B^+}$$ is called the **codomain** of $$\vrt{f^\sharp}{f}$$.

Now, we present the nondeterministic notion of dynamical systems.

> **Definition**
>
> A **possibilistic dynamical system** is a $$P$$-lens of the form
>
> $$\vrt{\mathit{nextState}}{\mathit{output}} : \vrt{\mathit{State}}{\mathit{State}} \leftrightarrows \vrt{\mathit{In}}{\mathit{Out}}$$
>
> That is, a dynamical system is a $$P$$-lens whose domain is an arena of the form $$\vrt{\mathit{State}}{\mathit{State}}$$ for some set $$\mathit{State}$$.

We must also update the definitions of our composition operators.

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

Elements of the first component $$\mathbf{2}^{\mathbf{w} \times \mathbf{h}}$$ are functions mapping each board cell location to $$0$$ if the cell contains no wall and $$1$$ if the cell contains a wall. Elements of the second component $$(\mathbf{w} \times \mathbf{h})^{\mathbf{n}}$$ map each robot identifier $$i$$ to the cell location at which robot $$i$$ is currently located.

Our environment takes steps of two types: receive and send. For send steps, we need to store a mapping from robot identifiers to booleans indicating whether the robot's most recent action succeeded. Thus we define

$$\mathit{StepType} \defeq 1 + \mathbf{2}^{\mathbf{n}}$$

where the left summand $$1$$ is used for receive steps and the right summand is for send steps.

The state of our environment $$\mathit{Env}$$ is then defined as:

$$\mathit{State}_{\mathit{Env}} \defeq \mathit{BoardState} \times \mathit{StepType}$$

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
  src="/assets/images/modelling-megazeux/stepper_2.drawio.png"
  style="margin-top: 30px; margin-bottom: 30px"
>
<figcaption>
Figure 1
</figcaption>
</figure>

To help explain the diagram above, we introduce a notational convenience. Elements of function spaces like $$\mathit{Out}_{\mathit{Env}} = (1 + \mathit{BoardState} \times \mathbf{2})^{\mathbf{n}}$$ are functions from robot identifiers (elements of $$\mathbf{n}$$) to output values. We can construct such a function using *lambda notation*: the expression $$\lambda i \in \mathbf{n} . (0, \ast)$$ denotes the constant function that always returns $$(0, \ast)$$ regardless of which robot identifier $$i$$ you pass in. If you're familiar with programming, this is analogous to anonymous functions like Python's `lambda i: (0, star)` or JavaScript's `i => (0, star)`.

The diagram includes two combinational lenses labeled $$\mathit{t2f}$$ and $$\mathit{f2t}$$, which convert between n-ary Cartesian products and function spaces. When the $$n$$ robots are composed in parallel, their collective output is an $$n$$-tuple (an element of the $$n$$-fold Cartesian product), but the environment expects a function from robot identifiers to actions. Similarly, the environment outputs a function mapping each robot to its board state and success value, but the parallel composition of robots expects an $$n$$-tuple. The $$\mathit{t2f}$$ lens converts a tuple $$(v_0, v_1, \ldots, v_{n-1})$$ to the function $$\lambda i \in \mathbf{n} . v_i$$, while $$\mathit{f2t}$$ converts a function $$\phi : \mathbf{n} \to X$$ to the tuple $$(\phi(0), \phi(1), \ldots, \phi(n-1))$$.

Note that, unlike the tic-tac-toe game stepper from the previous post, the MegaZeux stepper has no need for multiplexors and demultiplexors. In tic-tac-toe, the demultiplexor routes the board state to exactly one player per turn (based on whose turn it is), and the multiplexor collects moves from whichever player is active. In MegaZeux, all robots act simultaneously at each turn, so there is no selective routing.

## The Environment

Now that we've presented the structure of our possibilistic game stepper as a diagram, let's define the subsystem $$\mathit{Env}$$. It is defined as the possibilistic system

$$\vrt{\mathit{nextState}_{\mathit{Env}}}{\mathit{output}_{\mathit{Env}}} : \vrt{\mathit{State}_{\mathit{Env}}}{\mathit{State}_{\mathit{Env}}} \leftrightarrows \vrt{\mathit{In}_{\mathit{Env}}}{\mathit{Out}_{\mathit{Env}}}$$

The $$\mathit{output}_{\mathit{Env}}$$ function replicates the wall and robot positions to all robots and distributes to each robot the result of its last action:

$$\mathit{output}_{\mathit{Env}} : \mathbf{2}^{\mathbf{w} \times \mathbf{h}} \times (\mathbf{w} \times \mathbf{h})^{\mathbf{n}} \times (1 + \mathbf{2}^{\mathbf{n}}) \to (1 + \mathbf{2}^{\mathbf{w} \times \mathbf{h}} \times (\mathbf{w} \times \mathbf{h})^{\mathbf{n}} \times \mathbf{2})^{\mathbf{n}}$$

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

For any other combination of state and input (which--as violations of our environment's preconditions--are never intended to arise), we return the empty set:

$$\mathit{nextState}_{\mathit{Env}}(z, a) \defeq \emptyset$$

## The Robots

For each robot identifier $$i \in \mathbf{n}$$, we write $$\mathit{Robot}_i$$ for the possibilistic system corresponding to robot $$i$$.

We posit the existence of a set $$\Sigma_i$$ containing the *mental state* of robot $$i$$. A value of $$\Sigma_i$$ might contain a program counter for the script robot $$i$$ is currently running, and it might contain private data members as well. For distinct $$i,j \in \mathbf{n}$$ the sets $$\Sigma_i$$ and $$\Sigma_j$$ are not required to be equal and are typically distinct.

In addition to mental state, the state of robot $$i$$ also contains *administrative state*. The administrative state $$S$$, which is uniform across all robots, synchronizes the robot with the environment and stages actions to transmit to the environment.

$$S \defeq 1 + \mathit{Act}$$

The value $$(0,\ast) \in S$$ means that the environment is currently in a $$\mathit{Submit}$$ state. This means that the robots are currently receiving the board state and their success values. The value $$(1, a) \in S$$ means that the environment is currently in a $$\mathit{Receive}$$ state. The value $$a$$ is the action computed by the robot at the previous step, which has been stored in preparation for transmission at the current step.

Then the state set of robot $$i$$ is defined as:

$$\mathit{State}_{\mathit{Robot}_i} \defeq S \times \Sigma_i$$

Figure 1 above shows us the input and output sets shared by the $$n$$ robots:

$$\mathit{In}_{\mathit{Robot}_i} \defeq 1 + (\mathit{BoardState} \times \mathbf{2})$$

$$\mathit{Out}_{\mathit{Robot}_i} \defeq 1 + \mathit{Act}$$

Note that $$\mathit{Out}_{\mathit{Robot}_i} = S$$.

Now, we must define $$\mathit{Robot}_i$$ as a possibilistic lens:

$$\mathit{Robot}_i : \vrt{\mathit{State}_{\mathit{Robot}_i}}{\mathit{State}_{\mathit{Robot}_i}} \leftrightarrows \vrt{\mathit{In}_{\mathit{Robot}_i}}{\mathit{Out}_{\mathit{Robot}_i}} \defeq \vrt{nextState_{\mathit{Robot}_i}}{\mathit{output}_{\mathit{Robot}_i}}$$

where we define

$$\mathit{output}_{\mathit{Robot}_i} : S \times \Sigma_i \to 1 + \mathit{Act}$$

$$\mathit{output}_{\mathit{Robot}_i}(s, \sigma) \defeq s$$

and

$$\mathit{nextState}_{\mathit{Robot}_i} : S \times \Sigma_i \times (1 + \mathit{BoardState} \times \mathbf{2}) \to P(S \times \Sigma_i)$$

We posit the existence of a function $$\phi_i : \Sigma_i \times \mathit{BoardState} \times \mathbf{2} \to \mathit{Act} \times \Sigma_i$$ which, given robot $$i$$'s mental state, the current board state, and whether its last action succeeded, computes the next action and updates the mental state.

When in the "robot receiving" state $$(0, *)$$, the robot expects board state as input, and uses $$\phi$$ to compute its new state and action.

$$\mathit{nextState}_{\mathit{Robot}_i}((0, \ast), \sigma, (1, (b, s))) \defeq \{((1, a), \sigma')\}$$

$$\text{where } (a, \sigma') = \phi_i(\sigma, b, s)$$

When in the "robot sending" state $$(1, a)$$, the robot expects empty input and transitions back to the "robot receiving" state.

$$\mathit{nextState}_{\mathit{Robot}_i}((1, a), \sigma, (0, \ast)) \defeq \{((0, \ast), \sigma)\}$$

For any other combination of state and input (precondition violations), we return the empty set:

$$\mathit{nextState}_{\mathit{Robot}_i}(z, \sigma, x) \defeq \emptyset$$

## Wiring things up

We now have all the pieces to assemble our game stepper: the environment $$\mathit{Env}$$ and the $$n$$ robots $$\mathit{Robot}_0, \ldots, \mathit{Robot}_{n-1}$$. To create the full system, we need to connect the robots layer to the environment layer using composition.

The robots layer is formed by composing all robots in parallel:

$$\mathit{Robots} \defeq \mathit{Robot}_0 \otimes \mathit{Robot}_1 \otimes \cdots \otimes \mathit{Robot}_{n-1}$$

We convert to the environment's input set by post-composing by the $$\mathit{t2f}$$ combinational lens.

$$A \defeq \mathit{t2f} \circ \mathit{Robots}$$

We convert from the environment's output set to the players' input set by postcomposing with $$\mathit{f2t}$$

$$B \defeq \mathit{f2t} \circ \mathit{Env}$$

The signatures of $$A$$ and $$B$$ are

$$
A : \vrt{\mathit{State_{\mathit{Robot}_1}} \times \cdots \times \mathit{State}_{\mathit{Robot}_n}}{\mathit{State_{\mathit{Robot}_1}} \times \cdots \times \mathit{State}_{\mathit{Robot}_n}} \leftrightarrows \vrt{(1 + \mathit{BoardState} \times \mathbf{2}) \times \underset{n}{\cdots} \times (1 + \mathit{BoardState} \times \mathbf{2})}{(1 + \mathit{Act})^{\mathbf{n}}}
$$

and

$$
B : \vrt{\mathit{State_{\mathit{Env}}}}{\mathit{State}_{\mathit{Env}}} \leftrightarrows \vrt{(1 + \mathit{Act})^{\mathbf{n}}}{(1 + \mathit{BoardState} \times \mathbf{2}) \times \underset{n}{\cdots} \times (1 + \mathit{BoardState} \times \mathbf{2})}
$$

We juxtapose them with the parallel product:

$$A \otimes B : \vrt{\mathit{State_{\mathit{Robot}_1}} \times \cdots \times \mathit{State}_{\mathit{Robot}_n} \times \mathit{State}_{\mathit{Env}}}{\mathit{State_{\mathit{Robot}_1}} \times \cdots \times \mathit{State}_{\mathit{Robot}_n} \times \mathit{State}_{\mathit{Env}}} \leftrightarrows \vrt{(1 + \mathit{BoardState} \times \mathbf{2}) \times \underset{n}{\cdots} \times (1 + \mathit{BoardState} \times \mathbf{2}) \times (1 + \mathit{Act})^{\mathbf{n}}}{(1 + \mathit{Act})^{\mathbf{n}} \times (1 + \mathit{BoardState} \times \mathbf{2}) \times \underset{n}{\cdots} \times (1 + \mathit{BoardState} \times \mathbf{2})}$$

which is depicted by the following diagram

<figure>
<img
  src="/assets/images/modelling-megazeux/ab-juxtapose.drawio.png"
  style="margin-top: 30px; margin-bottom: 30px"
>
<figcaption>
Figure 2
</figcaption>
</figure>

We wire $$A$$ and $$B$$ together by post-composing $$A \otimes B$$ with a wiring lens that creates the feedback loop. The wiring lens must take the output of $$A \otimes B$$ (which has $$A$$'s output followed by $$B$$'s output) and route it appropriately: $$B$$'s output should feed into $$A$$'s input, and $$A$$'s output should feed into $$B$$'s input.

$$\vrt{w^\sharp}{w} : \vrt{(1 + \mathit{BoardState} \times \mathbf{2}) \times \underset{n}{\cdots} \times (1 + \mathit{BoardState} \times \mathbf{2}) \times (1 + \mathit{Act})^{\mathbf{n}}}{(1 + \mathit{Act})^{\mathbf{n}} \times (1 + \mathit{BoardState} \times \mathbf{2}) \times \underset{n}{\cdots} \times (1 + \mathit{BoardState} \times \mathbf{2})} \leftrightarrows \vrt{1}{1}$$

where the passforward function $$w$$ is defined as:

$$w(a, b_0, b_1, \ldots, b_{n-1}) \defeq \ast$$

This function simply discards both outputs, as the system is closed (there is no external environment to communicate with).

The passback function $$w^\sharp$$ creates the feedback connections:

$$w^\sharp(a, b_0, b_1, \ldots, b_{n-1}, \ast) \defeq \{(b_0, b_1, \ldots, b_{n-1}, a)\}$$

The complete game stepper is then:

$$\mathit{Stepper} \defeq \vrt{w^\sharp}{w} \circ (A \otimes B)$$

# Conclusion

Now we're getting somewhere. We have a game stepper that almost resembles MegaZeux. Like MegaZeux, it has a varying number of robots driven by distinct scripts. Also like MegaZeux, these robots are located on a 2D grid and move on the grid by submitting requests to an environment. Instead of executing the robot scripts in sequence, our model diverges from MegaZeux by executing the scripts of all robots concurrently at each frame. This more accurately reflects the real world and prevents a game from confusing developers with hidden execution ordering rules.

## Looking Ahead

Recall our original checklist of features:

* ✅ A robot performs physical actions by submitting requests to an environment. The environment decides whether to honor these requests.
* ✅ A robot has internal state, but it's a purely "mental" state that is used to make decisions. Physical properties of the robot are stored in the environment's internal state.
* ❌ A robot may send a message to another robot, which represents information sent along some physical communication medium.

We still need to implement the last one. Now that we've extended our dynamical systems formalism to *possibilistic* dynamical systems capable of modelling non-deterministic behavior, robots can process all messages received on the previous frame in any non-deterministically chosen order.

To refresh your memory, here is the sample pseudo-code of robot message passing that I presented in the previous post.

```
// When the player switches off the fuse box, this channel notifies all subscribers.
channel fuse_box_off : 1;

robot roy {
  receives on fuse_box_off;

  state working {
    handler fuse_box_off (_ : 1) = {
      set_state(fix_fusebox);
    }
  }

  state fix_fusebox {
    ...
  }
}

robot fuse_box {
  sends on fuse_box_off;

  state on {
    handler on_player_touch (_ : 1) = {
      send fuse_box_off(*);
      set_state(off);
    }
  }

  state off {
    ...
  }
}
```

Diagramatically, each robot $$R$$ appears as a rectangle representing a possibilistic dynamical system. Now, in addition to an input wire transmitting elements of the $$\mathit{GameState}$$ set, the system $$R$$ also has an input wire for each channel that it subscribes to as a receiver. In addition to the output wire transmitting elements of the set $$\mathit{Act}$$ of actions, it also has one output wire for each channel $$R$$ subscribes to as a sender.

So, for example, the $$\mathit{Roy}$$ system has two input wires: one transmits observations of $$\mathit{GameState}$$ sent from the environment, and the other transmits the collection of all messages sent along the $$\mathit{fuse\_box\_off}$$ channel during a single frame. Because $$\mathit{Roy}$$ does not subscribe to any channels as a sender, it has only an $$\mathit{Act}$$ output wire.

<figure>
<img
  src="/assets/images/modelling-megazeux/roy-example.drawio.png"
  style="margin-top: 30px; margin-bottom: 30px"
>
<figcaption>
Figure 3
</figcaption>
</figure>

And likewise for the $$\mathit{FuseBox}$$ system.

<figure>
<img
  src="/assets/images/modelling-megazeux/fuse-box-solo.drawio.png"
  style="margin-top: 30px; margin-bottom: 30px"
>
<figcaption>
Figure 4
</figcaption>
</figure>

Now, you might wonder why the channel wires are labelled with $$M_{\mathit{fin}} 1$$ even though the $$\mathit{fuse\_box\_off}$$ channel was declared with type $$1$$. The reason is that $$1$$ is the type of the individual messages that the channel transmits, but the channel may transmit multiple messages from multiple sender robots on the same frame. Because we've decided that distinct messages sent on the same frame are sent "at the same instant" (more precisely, they're sent within such a small interval that determining their order is too fine-grained to fall within the scope of our game system), the channel wire must transmit an unordered collection of messages rather than a single message. Given a set $$X$$, the elements of the set $$P X$$ are unordered collections of the set $$X$$. But in elements of $$P X$$ (subsets of $$X$$), all elements are unique. This is not appropriate for message collections, because multiple robots, or even a single robot, can transmit the same message multiple times along a channel on a single frame. Furthermore, if $$X$$ is an infinite set, say the set $$\mathbb N = \{ 0, 1, 2, \ldots \}$$ of natural numbers, then its subsets can be infinite as well; collections of messages must be finite so that a robot can sequentially process all of them.

> **Definition** Given a set $$X$$, the set $$M_{\text{fin}}X$$ of *finite multisets* of $$X$$ is defined as follows:
>
> $$M_{\text{fin}}X \defeq \{ f : X \to \mathbb{N} \mid f \text{ has finite support} \} $$
>
> where a function has finite support if $$\{x \in X \mid f(x) > 0\}$$ is finite. A function $$f : X \to \mathbb N$$ represents a multiset where each element $$x \in X$$ appears $$f(x)$$ times.

To leave things off, the following informal diagram demonstrates how message channels are wired together in the game stepper system.

<figure>
<img
  src="/assets/images/modelling-megazeux/channel-wiring.drawio.png"
  style="margin-top: 30px; margin-bottom: 30px"
>
<figcaption>
Figure 5
</figcaption>
</figure>

For each channel, all of the sender wires are directed into a combinational lens called $$\mathit{dist}$$. This lens takes the union of the multisets and duplicates this union along each of the wires going into the receivers of the channel. The receivers store the multiset internally, waiting for the next frame to process it and replace it. We'll cover how this works in more detail in a future post

