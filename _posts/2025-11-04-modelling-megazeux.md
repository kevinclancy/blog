---
layout: post_classic
title: Modelling MegaZeux
date: 2025-11-04 09:00 -0700
published: false
---

$$\newcommand{vrt}[2]{\left ( \begin{array}{l} #1 \\ #2 \end{array} \right )} $$
$$\newcommand{defeq}{\overset{\mathit{def}}{=}}$$

# Overview

Now that [we've modeled tic-tac-toe using dynamical systems]({% post_url 2025-10-18-modelling-tic-tac-toe %}), let's try modelling MegaZeux. Our goal isn't to recreate every feature of the MegaZeux game creation system, but to understand its essence. Let's explore what MegaZeux is *trying* to be, redesigning any features we find to be arbitrary or inelegant.

# Concurrency, MegaZeux, and the Real World

In tic-tac-toe, players take turns making moves. But MegaZeux is a simulation of the real world. In the real world, all agents act concurrently.

As it turns out, MegaZeux does *not* faithfully reflect the concurrent nature of the real world. Robots execute in a turn-based manner every frame. The order that the robots execute in is decided by their position on the grid. It starts in the top-left corner, updating every robot it encounters as it moves right across the top row. Then it iterates across the second row, then the third, until it reaches the bottom-right corner of the game board.

This is awkward. It's an arbitrary rule that a game developer needs to learn to explain the behavior of MegaZeux games. Consider two side-by-side robots. If both attempt walking one cell to the left on the same frame then both robots move. However, if both walk one cell to the right then only the right robot moves, because when the left robot walks right, it crashes into the right robot.

Our model, unlike MegaZeux, will allow all robots to act at once. The game stepper will evolve in the following fashion. First, it executes the initialization step:

* The environment sends the initial game state to the robots; the game state includes positions of all robots, as well as wall positions.

Then, it executes these two steps in a loop:

1. All robots concurrently use the received information from the environment to update their internal states and send their actions ($$\mathit{Stay}$$, $$\mathit{North}$$, $$\mathit{East}$$, $$\mathit{South}$$, $$\mathit{West}$$) to the environment.

2. The Environment executes the actions received from the robots to the best of its ability. If a robot proposes moving into a wall, off of the game board, or into a location where another robot currently is, then the move is not executed. If multiple robots attempt to move into the same cell then the environment randomly chooses one to move into the cell, committing the "losers" to their present positions. The environment sends the resulting state of the game board back to the robots, along with a boolean indicating to each robot whether its move succeeded.

## Non-deterministic lenses and dynamical systems

According to our plan, when multiple robots attempt moving into the same cell, we randomly choose one to succeed. But this requires a feature beyond the scope of the technique of lenses and dynamical systems described in the previous blog post.

We'll need to our definitions so that a dynamical system can update its state non-deterministically. Recall that if $$\mathit{X}$$ is a set then $$P X$$ is the set of all subsets of $$X$$, where $$P -$$ is called the powerset operator. If we think of a set as a collection of possibilities, then what we want is a new formulation of dynamical systems, where the passback function has the type

$$\mathit{State} \times \mathit{In} \to P \mathit{State}$$

Instead of producing a successor state as output, it produces the set of all possible successor states. In [Categorical Systems Theory](https://www.davidjaz.com/Papers/DynamicalBook.pdf), this is called a *possibilistic* system. Before we define possibilistic systems, we must first define a possibilistic version of the more general notion of a lens.

> **Definition**
>
> A **$$P$$-lens** $$\vrt{f^\sharp}{f} : \vrt{A^-}{A^+} \leftrightarrows \vrt{B^-}{B^+}$$ is a pair consisting of two functions:
> * A passforward function $$f : A^+ \to B^+$$
> * A passback function $$f^\sharp : A^+ \times B^- \to P A^-$$
>
> The arena $$\vrt{A^-}{A^+}$$ is called the **domain** of $$\vrt{f^\sharp}{f}$$ and the arena $$\vrt{B^-}{B^+}$$ is called the **codomain** of $$\vrt{f^\sharp}{f}$$.

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






