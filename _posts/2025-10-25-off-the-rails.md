---
layout: post_classic
title: "Interlude: Off the Rails"
date: 2025-10-25 09:00 -0700
published: false
---

$$\newcommand{vrt}[2]{\left ( \begin{array}{l} #1 \\ #2 \end{array} \right )} $$
$$\newcommand{defeq}{\overset{\mathit{def}}{=}}$$

# Train Engineering

I don't know anything about engineering trains, but I think the following observation is relevant to any sort of engineering, whether engineering a mathematical model of computer games or designing a complex software system: most engineering effort should be focused on avoiding disasters, not to respond to them.

Imagine you're engineering a train. If the train somehow falls off the rails then the passengers are in danger. One response to this concern might be to carefully design for several off-rails scenarios.

* If the train drives into a rocky area, it's very important that the train remains upright after hitting a large rock. This has all sorts of implications for the distribution of weight in the train.
* If the train drives off a cliff, the aerodynamics of the train must prevent if from spinning upside down. A system of parachutes may be installed to break its fall.
* If the train falls into water, the passengers are at risk of drowning, so a backup oxygen system may be included on the train.

But messing with the train's weight distribution and aerodynamics just to handle a disaster that we'd rather avoid entirely increases the complexity of our task. What's worse, it probably works against our goal of keeping the train on the tracks! Disaster concerns should not overshadow our engineering task; instead, they should be limited to basic precautions such as seatbelts, cushy walls, and the like.

# Disasters as Precondition Violations

In [Modelling Tic-Tac-Toe]({% post_url 2025-10-18-modelling-tic-tac-toe %}), I defined the $$\mathit{StepType}$$ component of the environment's state as the set:

$$\{ \mathit{ReceiveFrom}(0), \mathit{SubmitTo}(1), \mathit{ReceiveFrom}(1), \mathit{SubmitTo}(1), \mathit{IllegalState} \}$$

When you read that post, you probably understood that $$\mathit{IllegalState}$$ is used for "off rails" behavior and the other four elements are used for "on rails" behavior. The environment enters $$\mathit{IllegalState}$$ if either

* It receives a move while it's in a $$\mathit{SubmitTo}$$ state
* It receives an "empty input" $$(0, \ast)$$ while it's in a $$\mathit{ReceiveFrom}$$ state
* It's already in $$\mathit{IllegalState}$$

The idea is that we have engineered our dynamical system such that none of the above three scenarios ever arises given that our initial state is valid. They are *precondition violations*. The environment system's preconditions are

* Locations $$(1,\ell)$$ must only be provided as the first component of the input when in a $$\mathit{ReceiveFrom}$$ state
* An "Empty input" $$(0, \ast)$$ must only be provided as the first component of the input when in a $$\mathit{SubmitTo}$$ state
* The system must never be in $$\mathit{IllegalState}$$

Making both off rails and on rails states siblings in the sense that they are elements of the same set $$\mathit{StepType}$$ is obfuscating. Off rails states should be distinguished formally from on rails states to increase the clarity of our model.

More importantly, we are still required to define the values that $$\mathit{nextState}_{\mathit{Environment}}$$ outputs when it is fed $$\mathit{IllegalState}$$ as the second component of its input. This is a distraction.

We solve these issues by defining notions of *partial lenses* and *partial dynamical systems*.

# Partial Systems Theory

Hello

